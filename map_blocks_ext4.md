本文档记录`ext4`文件系统源码之中实现文件之中的逻辑地址与磁盘中物理地址映射功能的关键函数`ext4_ext_map_blocks`函数内容，`ext4`使用了B+树来实现这个映射。

### `ext4_ext_map_blocks`函数

```c
/*
 * Block allocation/map/preallocation routine for extents based files
 *
 *
 * Need to be called with
 * down_read(&EXT4_I(inode)->i_data_sem) if not allocating file system block
 * (ie, create is zero). Otherwise down_write(&EXT4_I(inode)->i_data_sem)
 *
 * return > 0, number of blocks already mapped/allocated
 *          if create == 0 and these are pre-allocated blocks
 *          	buffer head is unmapped
 *          otherwise blocks are mapped
 *
 * return = 0, if plain look up failed (blocks have not been allocated)
 *          buffer head is unmapped
 *
 * return < 0, error case.
 */
int ext4_ext_map_blocks(handle_t *handle, struct inode *inode,
			struct ext4_map_blocks *map, int flags)
{
	struct ext4_ext_path *path = NULL;
	struct ext4_extent newex, *ex, ex2;
	struct ext4_sb_info *sbi = EXT4_SB(inode->i_sb);
	ext4_fsblk_t newblock = 0, pblk;
	int err = 0, depth, ret;
	unsigned int allocated = 0, offset = 0;
	unsigned int allocated_clusters = 0;
	struct ext4_allocation_request ar;
	ext4_lblk_t cluster_offset;

	ext_debug(inode, "blocks %u/%u requested\n", map->m_lblk, map->m_len);
	trace_ext4_ext_map_blocks_enter(inode, map->m_lblk, map->m_len, flags);

	/* find extent for this block */
	path = ext4_find_extent(inode, map->m_lblk, NULL, 0);
	if (IS_ERR(path)) {
		err = PTR_ERR(path);
		path = NULL;
		goto out;
	}

	depth = ext_depth(inode);

	/*
	 * consistent leaf must not be empty;
	 * this situation is possible, though, _during_ tree modification;
	 * this is why assert can't be put in ext4_find_extent()
	 */
	if (unlikely(path[depth].p_ext == NULL && depth != 0)) {
		EXT4_ERROR_INODE(inode, "bad extent address "
				 "lblock: %lu, depth: %d pblock %lld",
				 (unsigned long) map->m_lblk, depth,
				 path[depth].p_block);
		err = -EFSCORRUPTED;
		goto out;
	}

	ex = path[depth].p_ext;
	if (ex) {
		ext4_lblk_t ee_block = le32_to_cpu(ex->ee_block);
		ext4_fsblk_t ee_start = ext4_ext_pblock(ex);
		unsigned short ee_len;


		/*
		 * unwritten extents are treated as holes, except that
		 * we split out initialized portions during a write.
		 */
		ee_len = ext4_ext_get_actual_len(ex);

		trace_ext4_ext_show_extent(inode, ee_block, ee_start, ee_len);

		/* if found extent covers block, simply return it */
		if (in_range(map->m_lblk, ee_block, ee_len)) {
			newblock = map->m_lblk - ee_block + ee_start;
			/* number of remaining blocks in the extent */
			allocated = ee_len - (map->m_lblk - ee_block);
			ext_debug(inode, "%u fit into %u:%d -> %llu\n",
				  map->m_lblk, ee_block, ee_len, newblock);

			/*
			 * If the extent is initialized check whether the
			 * caller wants to convert it to unwritten.
			 */
			if ((!ext4_ext_is_unwritten(ex)) &&
			    (flags & EXT4_GET_BLOCKS_CONVERT_UNWRITTEN)) {
				err = convert_initialized_extent(handle,
					inode, map, &path, &allocated);
				goto out;
			} else if (!ext4_ext_is_unwritten(ex)) {
				map->m_flags |= EXT4_MAP_MAPPED;
				map->m_pblk = newblock;
				if (allocated > map->m_len)
					allocated = map->m_len;
				map->m_len = allocated;
				ext4_ext_show_leaf(inode, path);
				goto out;
			}

			ret = ext4_ext_handle_unwritten_extents(
				handle, inode, map, &path, flags,
				allocated, newblock);
			if (ret < 0)
				err = ret;
			else
				allocated = ret;
			goto out;
		}
	}

	/*
	 * requested block isn't allocated yet;
	 * we couldn't try to create block if create flag is zero
	 */
	if ((flags & EXT4_GET_BLOCKS_CREATE) == 0) {
		ext4_lblk_t len;

		len = ext4_ext_determine_insert_hole(inode, path, map->m_lblk);

		map->m_pblk = 0;
		map->m_len = min_t(unsigned int, map->m_len, len);
		goto out;
	}

	/*
	 * Okay, we need to do block allocation.
	 */
	newex.ee_block = cpu_to_le32(map->m_lblk);
	cluster_offset = EXT4_LBLK_COFF(sbi, map->m_lblk);

	/*
	 * If we are doing bigalloc, check to see if the extent returned
	 * by ext4_find_extent() implies a cluster we can use.
	 */
	if (cluster_offset && ex &&
	    get_implied_cluster_alloc(inode->i_sb, map, ex, path)) {
		ar.len = allocated = map->m_len;
		newblock = map->m_pblk;
		goto got_allocated_blocks;
	}

	/* find neighbour allocated blocks */
	ar.lleft = map->m_lblk;
	err = ext4_ext_search_left(inode, path, &ar.lleft, &ar.pleft);
	if (err)
		goto out;
	ar.lright = map->m_lblk;
	err = ext4_ext_search_right(inode, path, &ar.lright, &ar.pright, &ex2);
	if (err < 0)
		goto out;

	/* Check if the extent after searching to the right implies a
	 * cluster we can use. */
	if ((sbi->s_cluster_ratio > 1) && err &&
	    get_implied_cluster_alloc(inode->i_sb, map, &ex2, path)) {
		ar.len = allocated = map->m_len;
		newblock = map->m_pblk;
		goto got_allocated_blocks;
	}

	/*
	 * See if request is beyond maximum number of blocks we can have in
	 * a single extent. For an initialized extent this limit is
	 * EXT_INIT_MAX_LEN and for an unwritten extent this limit is
	 * EXT_UNWRITTEN_MAX_LEN.
	 */
	if (map->m_len > EXT_INIT_MAX_LEN &&
	    !(flags & EXT4_GET_BLOCKS_UNWRIT_EXT))
		map->m_len = EXT_INIT_MAX_LEN;
	else if (map->m_len > EXT_UNWRITTEN_MAX_LEN &&
		 (flags & EXT4_GET_BLOCKS_UNWRIT_EXT))
		map->m_len = EXT_UNWRITTEN_MAX_LEN;

	/* Check if we can really insert (m_lblk)::(m_lblk + m_len) extent */
	newex.ee_len = cpu_to_le16(map->m_len);
	err = ext4_ext_check_overlap(sbi, inode, &newex, path);
	if (err)
		allocated = ext4_ext_get_actual_len(&newex);
	else
		allocated = map->m_len;

	/* allocate new block */
	ar.inode = inode;
	ar.goal = ext4_ext_find_goal(inode, path, map->m_lblk);
	ar.logical = map->m_lblk;
	/*
	 * We calculate the offset from the beginning of the cluster
	 * for the logical block number, since when we allocate a
	 * physical cluster, the physical block should start at the
	 * same offset from the beginning of the cluster.  This is
	 * needed so that future calls to get_implied_cluster_alloc()
	 * work correctly.
	 */
	offset = EXT4_LBLK_COFF(sbi, map->m_lblk);
	ar.len = EXT4_NUM_B2C(sbi, offset+allocated);
	ar.goal -= offset;
	ar.logical -= offset;
	if (S_ISREG(inode->i_mode))
		ar.flags = EXT4_MB_HINT_DATA;
	else
		/* disable in-core preallocation for non-regular files */
		ar.flags = 0;
	if (flags & EXT4_GET_BLOCKS_NO_NORMALIZE)
		ar.flags |= EXT4_MB_HINT_NOPREALLOC;
	if (flags & EXT4_GET_BLOCKS_DELALLOC_RESERVE)
		ar.flags |= EXT4_MB_DELALLOC_RESERVED;
	if (flags & EXT4_GET_BLOCKS_METADATA_NOFAIL)
		ar.flags |= EXT4_MB_USE_RESERVED;
	newblock = ext4_mb_new_blocks(handle, &ar, &err);
	if (!newblock)
		goto out;
	allocated_clusters = ar.len;
	ar.len = EXT4_C2B(sbi, ar.len) - offset;
	ext_debug(inode, "allocate new block: goal %llu, found %llu/%u, requested %u\n",
		  ar.goal, newblock, ar.len, allocated);
	if (ar.len > allocated)
		ar.len = allocated;

got_allocated_blocks:
	/* try to insert new extent into found leaf and return */
	pblk = newblock + offset;
	ext4_ext_store_pblock(&newex, pblk);
	newex.ee_len = cpu_to_le16(ar.len);
	/* Mark unwritten */
	if (flags & EXT4_GET_BLOCKS_UNWRIT_EXT) {
		ext4_ext_mark_unwritten(&newex);
		map->m_flags |= EXT4_MAP_UNWRITTEN;
	}

	err = ext4_ext_insert_extent(handle, inode, &path, &newex, flags);
	if (err) {
		if (allocated_clusters) {
			int fb_flags = 0;

			/*
			 * free data blocks we just allocated.
			 * not a good idea to call discard here directly,
			 * but otherwise we'd need to call it every free().
			 */
			ext4_discard_preallocations(inode, 0);
			if (flags & EXT4_GET_BLOCKS_DELALLOC_RESERVE)
				fb_flags = EXT4_FREE_BLOCKS_NO_QUOT_UPDATE;
			ext4_free_blocks(handle, inode, NULL, newblock,
					 EXT4_C2B(sbi, allocated_clusters),
					 fb_flags);
		}
		goto out;
	}

	/*
	 * Reduce the reserved cluster count to reflect successful deferred
	 * allocation of delayed allocated clusters or direct allocation of
	 * clusters discovered to be delayed allocated.  Once allocated, a
	 * cluster is not included in the reserved count.
	 */
	if (test_opt(inode->i_sb, DELALLOC) && allocated_clusters) {
		if (flags & EXT4_GET_BLOCKS_DELALLOC_RESERVE) {
			/*
			 * When allocating delayed allocated clusters, simply
			 * reduce the reserved cluster count and claim quota
			 */
			ext4_da_update_reserve_space(inode, allocated_clusters,
							1);
		} else {
			ext4_lblk_t lblk, len;
			unsigned int n;

			/*
			 * When allocating non-delayed allocated clusters
			 * (from fallocate, filemap, DIO, or clusters
			 * allocated when delalloc has been disabled by
			 * ext4_nonda_switch), reduce the reserved cluster
			 * count by the number of allocated clusters that
			 * have previously been delayed allocated.  Quota
			 * has been claimed by ext4_mb_new_blocks() above,
			 * so release the quota reservations made for any
			 * previously delayed allocated clusters.
			 */
			lblk = EXT4_LBLK_CMASK(sbi, map->m_lblk);
			len = allocated_clusters << sbi->s_cluster_bits;
			n = ext4_es_delayed_clu(inode, lblk, len);
			if (n > 0)
				ext4_da_update_reserve_space(inode, (int) n, 0);
		}
	}

	/*
	 * Cache the extent and update transaction to commit on fdatasync only
	 * when it is _not_ an unwritten extent.
	 */
	if ((flags & EXT4_GET_BLOCKS_UNWRIT_EXT) == 0)
		ext4_update_inode_fsync_trans(handle, inode, 1);
	else
		ext4_update_inode_fsync_trans(handle, inode, 0);

	map->m_flags |= (EXT4_MAP_NEW | EXT4_MAP_MAPPED);
	map->m_pblk = pblk;
	map->m_len = ar.len;
	allocated = map->m_len;
	ext4_ext_show_leaf(inode, path);
out:
	ext4_free_ext_path(path);

	trace_ext4_ext_map_blocks_exit(inode, flags, map,
				       err ? err : allocated);
	return err ? err : allocated;
}
```

函数参数之中`inode`用于描述文件系统之中文件或者目录的位置以及属性等信息，`map`之中保存查找的逻辑块范围、映射建立之后逻辑块对应的物理块范围等信息，代码之中`struct ext4_extent`类型实例保持了逻辑块范围与物理块范围的映射关系。`ext4`文件系统之中使用B+树来保存物理块与逻辑块的映射关系，这个函数以及被调用的函数中许多内容都涉及到了B+树的操作。

#### B+树中搜索

B+树中的叶子节点保存了文件逻辑块地址与物理块的映射关系，`ext4_find_extent`函数返回B+树之中一个最接近`map`中给定逻辑块起始地址的叶子节点，即这个叶子节点中保存的逻辑块地址与`map`中给定的逻辑块地址最接近。`ext4_find_extent`函数返回的`path`之中包含从B+树的根节点到这个叶子节点的每个节点，叶子节点为`extent`节点、其他的节点为索引节点，`extent`节点保存逻辑块地址与物理块地址映射关系、索引节点用于在B+树之中查找某个逻辑块地址。

在`ext4`文件系统中每个文件都有一个`inode`，一个`inode`之中有一棵B+树与之对应，这棵B+树的根节点存储在`struct ext4_inode_info`的`i_data`之中，`ext4_depth`获取到的值为这棵B+树的深度。代码之中还有一个关于异常情况的检查，如果B+树的深度不为0并且对应的`extent`为空意味着本应该找到B+树的一个叶子节点但是实际上没能找到，跳转到标签`out`处执行。

#### 处理最接近的叶子节点

这部分内容对应于`path[depth].p_ext`不为空时执行的代码，代码之中`ee_block`为逻辑块的起始地址、`ee_start`为逻辑块对应的物理块的起始地址、`ee_len`为映射的逻辑块/物理块的个数，通过`in_range`函数确认`map`指定的逻辑块范围是否在寻找到的`extent`节点保存的逻辑块范围之内，只有当`map`指定的逻辑块范围在`extent`节点保存的逻辑块范围内时运行接下来的操作。

`newblock`为`map`之中指定逻辑块起始地址对应的物理块起始地址，`allocated`初始化为`extent`节点`ex`之中在`map`指定的逻辑块起始地址之后的逻辑块数量。如果`ex`已经初始化并且允许将`extent`节点转换为未初始化状态，将`ex`重置为未初始化状态，随后跳转到标签`out`处执行，将一个`extent`节点重置成未初始化状态由`convert_initialized_extent`函数实现；如果`ex`已经初始化并且不允许将`extent`节点转换为未初始化状态，将查找到的物理块起始地址、映射的物理块个数写入到`map`之中，跳转到标签`out`处继续执行；若没有命中以上两种情况则意味着找到的`ex`还没有初始化，调用`ext4_ext_handle_unwritten_extents`对`ex`进行初始化，跳转到标签`out`处继续执行。

#### 未找到对应的物理块

这部分内容对应代码之中`if ((flags & EXT4_GET_BLOCKS_CREATE) == 0) {}`到标签`out`之间的代码，这部分内容对应未找到`map`之中指定逻辑块对应的物理块还未分配的情况。此时若不允许分配新的物理块，调用`ext4_ext_determine_insert_hole`函数确定还有多少物理块需要分配，因为有一些逻辑块已经分配了对应了的物理块，需要计算还有多少物理块需要分配而不是直接返回`map`之中需要指定的物理块个数，将还需要分配的物理块保存到`map`之中跳转到标签`out`处继续执行。

在`ext4`文件系统之中一个`cluster`由多个连续的块组成，当允许分配新的物理块时先检查`map`之中指定的逻辑块是否包含在一个`cluster`之中，这个判断由`get_implied_cluster_alloc`函数完成，当`map`之中指定的逻辑块包含在了一个`cluster`之中意味着已经为逻辑块分配了对应的物理块，跳转到标签`got_allocated_blocks`处继续执行。

当一个`cluster`之中的块无法满足需求，需要考虑分配新的物理块。

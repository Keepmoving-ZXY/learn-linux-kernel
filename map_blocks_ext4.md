本文档记录`ext4`文件系统源码之中实现文件之中的逻辑地址与磁盘中物理地址映射功能的关键函数`ext4_ext_map_blocks`函数内容，`ext4`使用了B+树来实现这个映射，文件系统之中基本存储单位为块、每个块的大小默认为4kb，文档之中提到的逻辑块为文件之中一个或多个连续块，物理块为磁盘之中一个或者多个连续块，大部分情况下逻辑块或物理块之中包含多个连续的块，只包含一个块的情况会有特殊说明。

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

函数参数之中`inode`用于描述文件系统之中文件或者目录的位置以及属性等信息，`map`之中保存查找的逻辑块范围、映射建立之后逻辑块对应的物理块范围等信息，代码之中`struct ext4_extent`类型实例保持了逻辑块范围与物理块范围的映射关系。`ext4`文件系统之中使用B+树来保存物理块与逻辑块的映射关系，这个函数以及被调用的函数中许多内容都涉及到了B+树的操作，B+树中的每个节点由一个`struct ext4_extent_header`的实例以及多个`struct ext4_extent_idx`实例或者`struct ext4_extent`实例组成，`struct ext4_extent_header`实例叫做节点头：当一个节点中由一个节点头以及多个`struct ext4_extent_idx`实例组成时这个节点叫做索引节点，一个`struct ext4_extent_idx`实例叫做索引，一个索引之中保存一个逻辑块与下一层节点的映射；当一个节点由一个节点头以及多个`struct ext4_extent`实例组成时这个节点叫做`extent`节点，一个`struct ext4_extent`实例叫做`extent`。B+树中索引节点保存接下来应该查找哪些节点，`extent`节点中保存逻辑块与物理块的映射关系。

#### B+树中搜索

B+树中的叶子节点保存了文件逻辑块地址与物理块的映射关系，`ext4_find_extent`函数返回B+树之中一个最接近`map`中给定逻辑块起始地址的`extent`，即这个`extent`中保存的逻辑块地址与`map`中给定的逻辑块地址最接近。`ext4_find_extent`函数返回的`path`之中包含从B+树的根节点到这个叶子节点的每个节点中的索引或者`extent`，`extent`节点保存逻辑块地址与物理块地址映射关系、索引节点用于在B+树之中查找某个逻辑块地址。

在`ext4`文件系统中每个文件都有一个`inode`，一个`inode`之中有一棵B+树与之对应，这棵B+树的根节点存储在`struct ext4_inode_info`的`i_data`之中，`ext4_depth`获取到的值为这棵B+树的深度。代码之中还有一个关于异常情况的检查，如果B+树的深度不为0并且对应的`extent`为空意味着本应该找到B+树的一个叶子节点但是实际上没能找到，跳转到标签`out`处执行。

#### 处理最接近的叶子节点

这部分内容对应于`path[depth].p_ext`不为空时执行的代码，代码之中`ee_block`为逻辑块的起始地址、`ee_start`为逻辑块对应的物理块的起始地址、`ee_len`为映射的逻辑块/物理块的个数，通过`in_range`函数确认`map`指定的逻辑块范围是否在寻找到的`extent`节点保存的逻辑块范围之内，只有当`map`指定的逻辑块范围在`extent`节点保存的逻辑块范围内时运行接下来的操作。

`newblock`为`map`之中指定逻辑块起始地址对应的物理块起始地址，`allocated`初始化为`extent`节点`ex`之中在`map`指定的逻辑块起始地址之后的逻辑块数量。如果`ex`已经初始化并且允许将`extent`节点转换为未初始化状态，将`ex`重置为未初始化状态，随后跳转到标签`out`处执行，将一个`extent`节点重置成未初始化状态由`convert_initialized_extent`函数实现；如果`ex`已经初始化并且不允许将`extent`节点转换为未初始化状态，将查找到的物理块起始地址、映射的物理块个数写入到`map`之中，跳转到标签`out`处继续执行；若没有命中以上两种情况则意味着找到的`ex`还没有初始化，调用`ext4_ext_handle_unwritten_extents`对`ex`进行初始化，跳转到标签`out`处继续执行。

#### 未找到对应的物理块

这部分内容对应代码之中`if ((flags & EXT4_GET_BLOCKS_CREATE) == 0) {}`到标签`out`之间的代码，这部分内容对应未找到`map`之中指定逻辑块对应的物理块还未分配的情况。此时若不允许分配新的物理块，调用`ext4_ext_determine_insert_hole`函数确定还有多少物理块需要分配，因为有一些逻辑块已经分配了对应了的物理块，需要计算还有多少物理块需要分配而不是直接返回`map`之中需要指定的物理块个数，将还需要分配的物理块保存到`map`之中跳转到标签`out`处继续执行。

在`ext4`文件系统之中一个`cluster`由多个连续的块组成，当允许分配新的物理块时先检查`map`之中指定的逻辑块是否包含在一个`cluster`之中，这个判断由`get_implied_cluster_alloc`函数完成，当`map`之中指定的逻辑块包含在了一个`cluster`之中意味着已经为逻辑块分配了对应的物理块，跳转到标签`got_allocated_blocks`处继续执行。

当一个`cluster`之中的块无法满足需求，需要分配新的物理块，新的物理块分配流程为:1).找到这个待分配的逻辑块在B+树之中左右相邻的逻辑块，使用`ext4_ext_search_left`函数找到左边相邻的逻辑块，使用`ext4_ext_search_right`找到右边相邻的逻辑块，左边相邻的逻辑块意味着逻辑块起始地址小于待分配逻辑块起始地址并且与待分配逻辑块起始地址最接近，右边相邻的逻辑块意味着逻辑块起始地址大于待分配逻辑块并且与待分配逻辑块起始地址最接近；2).考虑直接分配一个簇用于满足逻辑块的分配请求，当右边临近的逻辑块未找到(`ext4_ext_search_right`返回值不为0)时调用`get_implied_cluster_alloc`尝试直接在待分配逻辑块地址附近直接分配一个新的簇，若分配成功跳转到标签`got_allocated_blocks`处继续执行；3).检查待分配的逻辑块数量是否会超过单个`extent`映射的块数量上限，如果超过则将块数量重置为单个`extent`映射的块数量上限，从代码之中可以看到当`extent`已经被初始化时、未被初始化时能够映射的块数量上限不同，这是因为代码之中使用了`extent`之中的`ee_len`的`MSB`来表示`extent`是否初始化，详见`EXT_INIT_MAX_LEN、EXT_UNWRITTEN_MAX_LEN`两个宏定义前的注释；4).检查待分配的逻辑块是否与现有的某个`extent`之中的逻辑块有重叠，重叠检查使用`ext4_ext_check_overlap`函数完成，如果有重叠调用`ext4_ext_get_actual_len`计算未重叠部分块的数量，将需要分配的物理块数量存入`allocated`变量之中；5).计算待分配的逻辑块起始地址对应物理块起始地址、在`cluster`之中的偏移量、需要分配的`cluster`个数，将逻辑块起始地址、物理块起始地址对其到`cluster`的起始块位置，设置分配标记位以在分配过程中执行特定的操作；6).调用`ext4_mb_new_blocks`分配物理块，这个函数返回物理块的起始地址，如果返回0意味着分配过程中出现错误跳转到标签`out`处继续执行，分配成功则在`ar`的`len`字段之中保存建立`map`之中逻辑块与物理块映射所需的物理块数量，这里需要注意的是`len`的值可能和实际分配的物理块个数不一致，因为分配是以`cluster`为单位的，需要分配的物理块数量可能不会占据分配到的`cluster`中的所有的物理块，至此物理块分配流程结束。

#### 标签`got_allocated_blocks`

这部分内容对应代码中标签`got_allocated_blocks`中的代码，`newex`保存新创建的`extent`的内容，主要的流程为：1).向`newex`之中填入物理块起始地址(`ext4_ext_store_pblock`函数)、分配的物理块数量，若需要将其设置为未初始化状态则调用`ext4_ext_mark_unwritten`函数进行转换；2).将新创建的`extent`插入到B+树之中，对应的函数是`ext4_ext_insert_extent`，若插入失败丢弃已经预分配的物理块(`ext4_discard_preallocations`函数)、释放已经分配的物理块(`ext4_free_blocks`函数)、跳转到`out`处继续执行；3).在启用了延迟分配并且之前的流程中分配了新的`cluster`时调用`ext4_da_update_reserve_space`函数更新延迟分配机制中保留的`cluster`计数，其他情况下调用`ext4_es_delayed_clu`确认是否有延迟分配的`cluster`，若存在调用`ext4_da_update_reserve_space`更新保留`cluster`的计数。这里涉及到了`ext4`文件系统的延迟分配功能，这个功能实现当IO数据从缓存写入到磁盘之中时才会分配物理块而非在数据写入文件时立刻分配物理块，`ext4`文件系统之中会记录为延迟分配保留的`cluster`数量；4).更新inode同步事务，涉及到的函数为`ext4_update_inode_fsync_trans`；5).填充映射结果至`map`之中，其中`m_pblk`保存逻辑块映射的物理块起始地址、`m_len`保存映射的物理块数量。

#### 标签`out`

这部分内容对应代码之中标签`out`处的代码，这部分内容释放`ext4_find_extent`函数分配的资源，即调用`ext4_free_ext_path`函数释放`path`指向的结构，根据分配过程中是否发生错误确定返回值：若发生错误返回错误对应的错误代码(存储在`err`之中)，若未发生错误返回建立与逻辑块映射所需物理块数量。

### `ext4_find_extent`函数

```c
struct ext4_ext_path *
ext4_find_extent(struct inode *inode, ext4_lblk_t block,
		 struct ext4_ext_path **orig_path, int flags)
{
	struct ext4_extent_header *eh;
	struct buffer_head *bh;
	struct ext4_ext_path *path = orig_path ? *orig_path : NULL;
	short int depth, i, ppos = 0;
	int ret;
	gfp_t gfp_flags = GFP_NOFS;

	if (flags & EXT4_EX_NOFAIL)
		gfp_flags |= __GFP_NOFAIL;

	eh = ext_inode_hdr(inode);
	depth = ext_depth(inode);
	if (depth < 0 || depth > EXT4_MAX_EXTENT_DEPTH) {
		EXT4_ERROR_INODE(inode, "inode has invalid extent depth: %d",
				 depth);
		ret = -EFSCORRUPTED;
		goto err;
	}

	if (path) {
		ext4_ext_drop_refs(path);
		if (depth > path[0].p_maxdepth) {
			kfree(path);
			*orig_path = path = NULL;
		}
	}
	if (!path) {
		/* account possible depth increase */
		path = kcalloc(depth + 2, sizeof(struct ext4_ext_path),
				gfp_flags);
		if (unlikely(!path))
			return ERR_PTR(-ENOMEM);
		path[0].p_maxdepth = depth + 1;
	}
	path[0].p_hdr = eh;
	path[0].p_bh = NULL;

	i = depth;
	if (!(flags & EXT4_EX_NOCACHE) && depth == 0)
		ext4_cache_extents(inode, eh);
	/* walk through the tree */
	while (i) {
		ext_debug(inode, "depth %d: num %d, max %d\n",
			  ppos, le16_to_cpu(eh->eh_entries), le16_to_cpu(eh->eh_max));

		ext4_ext_binsearch_idx(inode, path + ppos, block);
		path[ppos].p_block = ext4_idx_pblock(path[ppos].p_idx);
		path[ppos].p_depth = i;
		path[ppos].p_ext = NULL;

		bh = read_extent_tree_block(inode, path[ppos].p_idx, --i, flags);
		if (IS_ERR(bh)) {
			ret = PTR_ERR(bh);
			goto err;
		}

		eh = ext_block_hdr(bh);
		ppos++;
		path[ppos].p_bh = bh;
		path[ppos].p_hdr = eh;
	}

	path[ppos].p_depth = i;
	path[ppos].p_ext = NULL;
	path[ppos].p_idx = NULL;

	/* find extent */
	ext4_ext_binsearch(inode, path + ppos, block);
	/* if not an empty leaf */
	if (path[ppos].p_ext)
		path[ppos].p_block = ext4_ext_pblock(path[ppos].p_ext);

	ext4_ext_show_path(inode, path);

	if (orig_path)
		*orig_path = path;
	return path;

err:
	ext4_free_ext_path(path);
	if (orig_path)
		*orig_path = NULL;
	return ERR_PTR(ret);
}
```

这个函数用于寻找B+树中与参数`block`指定的逻辑块最近的`extent`，即B+树中某个`extent`之中映射的逻辑块起始地址与`block`给定的逻辑块起始地址最接近，具体流程如下：

1).B+树深度检测，`depth`为B+树的深度，在开始搜索之前对B+树深度进行校验：若`depth`小于0或者大于B+树最大的深度跳转到标签`out`处执行；

2).已经存在的`struct ext4_ext_path`数组处理，  `struct ext4_ext_path`结构存储查找结果之中B+树每一层的索引节点或者叶子节点内容，`path`可能指向一个已经存在的数组，这个实例是之前某次搜索B+树时创建的，这种情况下调用`ext4_ext_drop_refs`函数释放B+树每一层的索引节点或者叶子节点占用的缓冲区，若之前搜索的B+树深度小于马上搜索的B+树的深度意味着这个实例无法容纳新的B+树中搜索结果，释放这个数组；

3).创建新的`struct ext4_ext_path`数组，当在之前的流程中传入的数组被释放或者没有传入数组的时候会创建新的数组，数组之中保存遍历到的B+树每一层的索引或者叶子节点，因此数组的长度要大于B+树的深度；

4).初始化`struct ext4_ext_path`数组，数组中第一个位置存储B+树根节点的节点头，当B+树只有一个根节点时调用`ext_cache_extents`将B+树的状态保存到`extent`状态树中；

5).逐层搜索B+树至倒数第二层，对于B+树的每一层调用`ext4_ext_binsearch_idx`函数使用二分查找这一层中在多个索引之中找到一个索引(这个索引之中保存的逻辑块是最小并且包含待查找逻辑块起始地址)、在`struct ext4_ext_path`数组之中当前层对应的位置保存这一层中找到的索引以及索引所在的节点头；

6).搜索B+树最后一层，这层之中的节点中存储的都是`extent`，调用`ext4_ext_binsearch`从这一层之中查找符合要求的`extent`并写入到`struct ext4_ext_path`数组之中的最后一个位置，符合要求的`extent`为保存的逻辑块之中最接近待查找的逻辑块起始地址的`extent`，返回`struct ext4_ext_path`数组。

7).标签`out`处的代码释放已经分配的`struct ext4_ext_path`数组，返回错误代码。

### `ext4_ext_determine_insert_hole`函数

```c
/*
 * Determine hole length around the given logical block, first try to
 * locate and expand the hole from the given @path, and then adjust it
 * if it's partially or completely converted to delayed extents, insert
 * it into the extent cache tree if it's indeed a hole, finally return
 * the length of the determined extent.
 */
static ext4_lblk_t ext4_ext_determine_insert_hole(struct inode *inode,
						  struct ext4_ext_path *path,
						  ext4_lblk_t lblk)
{
	ext4_lblk_t hole_start, len;
	struct extent_status es;

	hole_start = lblk;
	len = ext4_ext_find_hole(inode, path, &hole_start);
again:
	ext4_es_find_extent_range(inode, &ext4_es_is_delayed, hole_start,
				  hole_start + len - 1, &es);
	if (!es.es_len)
		goto insert_hole;

	/*
	 * There's a delalloc extent in the hole, handle it if the delalloc
	 * extent is in front of, behind and straddle the queried range.
	 */
	if (lblk >= es.es_lblk + es.es_len) {
		/*
		 * The delalloc extent is in front of the queried range,
		 * find again from the queried start block.
		 */
		len -= lblk - hole_start;
		hole_start = lblk;
		goto again;
	} else if (in_range(lblk, es.es_lblk, es.es_len)) {
		/*
		 * The delalloc extent containing lblk, it must have been
		 * added after ext4_map_blocks() checked the extent status
		 * tree, adjust the length to the delalloc extent's after
		 * lblk.
		 */
		len = es.es_lblk + es.es_len - lblk;
		return len;
	} else {
		/*
		 * The delalloc extent is partially or completely behind
		 * the queried range, update hole length until the
		 * beginning of the delalloc extent.
		 */
		len = min(es.es_lblk - hole_start, len);
	}

insert_hole:
	/* Put just found gap into cache to speed up subsequent requests */
	ext_debug(inode, " -> %u:%u\n", hole_start, len);
	ext4_es_insert_extent(inode, hole_start, len, ~0, EXTENT_STATUS_HOLE);

	/* Update hole_len to reflect hole size after lblk */
	if (hole_start != lblk)
		len -= lblk - hole_start;

	return len;
}
```

这个函数计算给定逻辑块(单个块)附近空洞的长度、将空洞对应的逻辑块标记位空洞状态放入到状态树之中、返回已经预留的的逻辑块长度，空洞为给定逻辑块(单个块)附近未映射物理块部分的长度，空洞不包含延迟分配的逻辑块，这个函数的流程如下：

1).计算逻辑块(单个块)附近空洞的长度，空洞的长度由`ext4_ext_find_hole`函数给出， 这个函数比较逻辑块(单个快)与遍历B+树得到的`extent`之中逻辑块地址得到；

2).获取空洞附近延迟分配的逻辑块，代码之中`hole_start`为空洞的起始地址、`len`为空洞的长度，通过`ext4_es_find_extent_range`函数获取空洞附近延迟分配的逻辑块，`ext4_es_find_extent_range`函数被调用的位置也是标签`again`所在的位置；

3).若状态树之中不包含空洞的状态，跳转到标签`insert_hole`处将空洞插入到状态树之中；

4).若状态树之中包含空洞的状态，则意味着空洞附近包含了一个延迟分配的逻辑块，根据空洞与延迟分配逻辑块的位置关系进行处理：

4.1).当给定逻辑块(单个块)位于延迟分配的逻辑块之后， 将空洞的起始地址设置为给定逻辑块(单个块)、空洞长度减少`lblk - hole_start`个逻辑块，跳转到标签`again`处继续搜索新的空洞附近延迟分配的逻辑块，在进行新的搜索之前空洞的起始地址更靠近空洞的结束地址，因此要减少空洞的长度；

4.2).当给定逻辑块(单个快)位于延迟分配的逻辑块之中，这意味着给定逻辑块(单个块)以及后边几个连续的逻辑块(单个块)已经被预留了，返回被预留的这几个逻辑块数量，注意此时返回的逻辑块数量时预留的、还没有分配对应的物理块；

4.3).当给定逻辑块(单个块)位于延迟分配的逻辑块之前，若空洞完全位于延迟分配的逻辑块之前保持`len`不变，若空洞与延迟分配的逻辑块有重叠设置`len`为空洞与延迟分配逻辑块起始地址之间的逻辑块个数，无论那种情况跳转到标签`insert_hole`继续执行；

5).标签`insert_hole`之中的代码将空洞插入到状态树之中，计算空洞之中位于给定逻辑块(单个块)之后的逻辑块数量并返回；

这个函数的返回值为已经预留的逻辑块长度，预留的逻辑块可能是空洞之中给定逻辑块(单个块)后边的部分，也可能是延迟分配的逻辑块之中给定逻辑块(单个块)后边的部分。

### `get_implied_cluster_alloc`函数

```c
/*
 * get_implied_cluster_alloc - check to see if the requested
 * allocation (in the map structure) overlaps with a cluster already
 * allocated in an extent.
 *	@sb	The filesystem superblock structure
 *	@map	The requested lblk->pblk mapping
 *	@ex	The extent structure which might contain an implied
 *			cluster allocation
 *
 * This function is called by ext4_ext_map_blocks() after we failed to
 * find blocks that were already in the inode's extent tree.  Hence,
 * we know that the beginning of the requested region cannot overlap
 * the extent from the inode's extent tree.  There are three cases we
 * want to catch.  The first is this case:
 *
 *		 |--- cluster # N--|
 *    |--- extent ---|	|---- requested region ---|
 *			|==========|
 *
 * The second case that we need to test for is this one:
 *
 *   |--------- cluster # N ----------------|
 *	   |--- requested region --|   |------- extent ----|
 *	   |=======================|
 *
 * The third case is when the requested region lies between two extents
 * within the same cluster:
 *          |------------- cluster # N-------------|
 * |----- ex -----|                  |---- ex_right ----|
 *                  |------ requested region ------|
 *                  |================|
 *
 * In each of the above cases, we need to set the map->m_pblk and
 * map->m_len so it corresponds to the return the extent labelled as
 * "|====|" from cluster #N, since it is already in use for data in
 * cluster EXT4_B2C(sbi, map->m_lblk).	We will then return 1 to
 * signal to ext4_ext_map_blocks() that map->m_pblk should be treated
 * as a new "allocated" block region.  Otherwise, we will return 0 and
 * ext4_ext_map_blocks() will then allocate one or more new clusters
 * by calling ext4_mb_new_blocks().
 */
static int get_implied_cluster_alloc(struct super_block *sb,
				     struct ext4_map_blocks *map,
				     struct ext4_extent *ex,
				     struct ext4_ext_path *path)
{
	struct ext4_sb_info *sbi = EXT4_SB(sb);
	ext4_lblk_t c_offset = EXT4_LBLK_COFF(sbi, map->m_lblk);
	ext4_lblk_t ex_cluster_start, ex_cluster_end;
	ext4_lblk_t rr_cluster_start;
	ext4_lblk_t ee_block = le32_to_cpu(ex->ee_block);
	ext4_fsblk_t ee_start = ext4_ext_pblock(ex);
	unsigned short ee_len = ext4_ext_get_actual_len(ex);

	/* The extent passed in that we are trying to match */
	ex_cluster_start = EXT4_B2C(sbi, ee_block);
	ex_cluster_end = EXT4_B2C(sbi, ee_block + ee_len - 1);

	/* The requested region passed into ext4_map_blocks() */
	rr_cluster_start = EXT4_B2C(sbi, map->m_lblk);

	if ((rr_cluster_start == ex_cluster_end) ||
	    (rr_cluster_start == ex_cluster_start)) {
		if (rr_cluster_start == ex_cluster_end)
			ee_start += ee_len - 1;
		map->m_pblk = EXT4_PBLK_CMASK(sbi, ee_start) + c_offset;
		map->m_len = min(map->m_len,
				 (unsigned) sbi->s_cluster_ratio - c_offset);
		/*
		 * Check for and handle this case:
		 *
		 *   |--------- cluster # N-------------|
		 *		       |------- extent ----|
		 *	   |--- requested region ---|
		 *	   |===========|
		 */

		if (map->m_lblk < ee_block)
			map->m_len = min(map->m_len, ee_block - map->m_lblk);

		/*
		 * Check for the case where there is already another allocated
		 * block to the right of 'ex' but before the end of the cluster.
		 *
		 *          |------------- cluster # N-------------|
		 * |----- ex -----|                  |---- ex_right ----|
		 *                  |------ requested region ------|
		 *                  |================|
		 */
		if (map->m_lblk > ee_block) {
			ext4_lblk_t next = ext4_ext_next_allocated_block(path);
			map->m_len = min(map->m_len, next - map->m_lblk);
		}

		trace_ext4_get_implied_cluster_alloc_exit(sb, map, 1);
		return 1;
	}

	trace_ext4_get_implied_cluster_alloc_exit(sb, map, 0);
	return 0;
}
```

这个函数确定`map`给定的逻辑块是否与某个已经分配的`cluster`有重叠，这个函数返回1时意味着`map`返回的逻辑块为`cluster`之中未被占用的逻辑块、返回0意味着需要分配新的`cluster`以满足`map`之中给定的逻辑块分配请求，`ee_block`、`ee_start`、`ee_len`为给定`extent`之中保存的逻辑块起始地址、物理块起始地址、物理块的长度(同时也是逻辑块的长度)，`rr_cluster_start`为`map`之中给定逻辑块所在的`cluster`，`ex`给定的`extent`之中保存的逻辑块可能会跨越多个`cluster`，`ex_cluster_start`为`ex`给定的`extent`之中逻辑块起始地址所在的`cluster`、`ex_cluster_end`为`ex`给定的`extent`之中逻辑块结束地址所在的`cluster`。`rr_cluster_start`与`ex_cluster_start`相同意味着`map`之中给定逻辑块起始地址与`ex`给定的`extent`之中保存的逻辑块起始地址属于同一个`cluster`，考虑到运行此函数的时候两个逻辑块没有重叠，可以推断出`map`之中给定逻辑块位于`ex`给定的`extent`之中保存逻辑块之前；`rr_cluster_start`与`ex_cluster_end`相同意味着`map`之中给定逻辑块起始地址与`ex`给定的`extent`之中保存的逻辑块结束地址属于同一个`cluster`，考虑到运行此函数的时候两个逻辑块没有重叠，可以推断出`map`之中给定逻辑块位于`ex`给定的`extent`之中保存的逻辑块之后。当`map`之中给定逻辑块没有与`ex`给定的`extent`在同一个`cluster`中时，返回0表明需要分配新的`cluster`。

当`map`之中给定逻辑块与`ex`给定的`extent`在同一个`cluster`之中时，函数的逻辑主要集中在`map`之中保存的物理块起始地址(`m_pblk`字段)以及物理块长度(`m_len`)的设置上，这两个值确定了`cluster`之中一个未被占用的物理块，返回1表示出现此种情况。`map`之中物理块起始地址需要考虑两种情况：当`map`之中给定的逻辑块在`ex`给定的`extent`中保存的逻辑块之后时，`map`之中给定的逻辑块部分或全部在`ex`给定的`extent`之中保存逻辑块占用的最后一个`cluster`之中，这个`cluster`的起始物理块地址加上`map`之中给定逻辑块在`cluster`之中的偏移得到了`map`之中给定逻辑块起始地址映射的物理块起始地址，这段逻辑对应的代码之中`ee_start`增加了`len-1`之后成为`ex`给定的`extent`之中保存的物理块中最后一个块的地址、`EXT4_PBLK_CMASK(sbi, ee_start)`为`ex`给定的`extent`之中保存逻辑块占用的最后一个`cluster`、`c_offset`为`map`之中给定的逻辑块起始地址在`cluster`之中的偏移；当`map`之中给定逻辑块在`ex`给定的`extent`中保存的逻辑块之前时，`map`之中给定的逻辑块部分或全部在`ex`给定的`extent`之中国保存逻辑块占用的第一个`cluster`之中，这个`cluster`的起始物理块地址加上`map`之中给定逻辑块在`cluster`之中的偏移得到了`map`之中给定逻辑块起始地址映射的物理块起始地址，这段逻辑对应的代码之中`ee_start`为`ex`给定的`extent`之中保持物理块的起始地址、`EXT4_PBLK_CMASK(sbi, ee_start)`为`ex`给定的`extent`之中保存逻辑块占用的第一个`cluster`，`c_offset`为`map`之中给定的逻辑块起始地址在`cluster`之中的偏移。`map`之中物理块长度保证物理块(由`map`之中物理块起始地址与长度确定)不会越过所在的`cluster`，代码之中使用`min(map->m_len, (unsigned) sbi->s_cluster_ratio - c_offset)`来保证这一点，接下来根据`map`之中保存的逻辑块起始地址、`ex`给定的`extent`之中保存的逻辑块起始地址的关系进一步调整`map`之中保存的物理块长度以保证`map`中保存的物理块未被占用：当`map`中给定的逻辑块起始地址小于`ex`给定的`extent`之中保存的逻辑块起始地址时，确保`map`之中保存的物理块长度最多为`cluster`之中这两个逻辑块起始地址对应的物理块(单个块)之间的的物理块个数；反之获取`ex`给定的`extent`之后的`extent`，`map`中给定的逻辑块起始地址与之后的`ex`给定的`extent`中保存的起始地址之间的物理块未被占用，确保`map`之中保存的物理块长度最多为这两个逻辑块起始地址对应的物理块(单个块)之间的物理块个数。

### `ext4_ext_search_left`函数

```c
/*
 * search the closest allocated block to the left for *logical
 * and returns it at @logical + it's physical address at @phys
 * if *logical is the smallest allocated block, the function
 * returns 0 at @phys
 * return value contains 0 (success) or error code
 */
static int ext4_ext_search_left(struct inode *inode,
				struct ext4_ext_path *path,
				ext4_lblk_t *logical, ext4_fsblk_t *phys)
{
	struct ext4_extent_idx *ix;
	struct ext4_extent *ex;
	int depth, ee_len;

	if (unlikely(path == NULL)) {
		EXT4_ERROR_INODE(inode, "path == NULL *logical %d!", *logical);
		return -EFSCORRUPTED;
	}
	depth = path->p_depth;
	*phys = 0;

	if (depth == 0 && path->p_ext == NULL)
		return 0;

	/* usually extent in the path covers blocks smaller
	 * then *logical, but it can be that extent is the
	 * first one in the file */

	ex = path[depth].p_ext;
	ee_len = ext4_ext_get_actual_len(ex);
	if (*logical < le32_to_cpu(ex->ee_block)) {
		if (unlikely(EXT_FIRST_EXTENT(path[depth].p_hdr) != ex)) {
			EXT4_ERROR_INODE(inode,
					 "EXT_FIRST_EXTENT != ex *logical %d ee_block %d!",
					 *logical, le32_to_cpu(ex->ee_block));
			return -EFSCORRUPTED;
		}
		while (--depth >= 0) {
			ix = path[depth].p_idx;
			if (unlikely(ix != EXT_FIRST_INDEX(path[depth].p_hdr))) {
				EXT4_ERROR_INODE(inode,
				  "ix (%d) != EXT_FIRST_INDEX (%d) (depth %d)!",
				  ix != NULL ? le32_to_cpu(ix->ei_block) : 0,
				  le32_to_cpu(EXT_FIRST_INDEX(path[depth].p_hdr)->ei_block),
				  depth);
				return -EFSCORRUPTED;
			}
		}
		return 0;
	}

	if (unlikely(*logical < (le32_to_cpu(ex->ee_block) + ee_len))) {
		EXT4_ERROR_INODE(inode,
				 "logical %d < ee_block %d + ee_len %d!",
				 *logical, le32_to_cpu(ex->ee_block), ee_len);
		return -EFSCORRUPTED;
	}

	*logical = le32_to_cpu(ex->ee_block) + ee_len - 1;
	*phys = ext4_ext_pblock(ex) + ee_len - 1;
	return 0;
}
```

这个函数搜索在`*logic`给定逻辑块左侧并且距离最近的已分配逻辑块(单个块)以及这个逻辑块(单个块)对应的物理块，搜索到的逻辑块(单个块)通过`logical`参数返回、物理块通过`phys`参数返回。

这个函数先进行基本的参数检查，当传入的`path`为空指针时直接返回错误码，当文件对应的B+树为空时设置返回的物理块为0并返回0，当`*logic`给定的逻辑块位于`ex`给定的`extent`保存的逻辑块范围之内时同样直接返回错误码。

由于B+树中`extent`查找时通过二分搜索算法进行的，这个算法实现保证只要存在某个`extent`之中保存的逻辑块起始地址小于`*logical`给定的逻辑块起始地址就一定会返回这个`extent`，因此当`*logical`给定的逻辑块起始地址小于`ex`给定的`extent`之中保存的逻辑块起始地址时意味着`ex`给定的`extent`是文件对应的B+树中的第一个`extent`。此时`path`之中保存的每层索引或者`extent`在对应节点中的第一个位置，代码之中使用`EXT_FIRST_EXTENT`以及`EXT_FIRST_INDEX`来检测当前`path`是否存在不满足此要求的`extent`或者索引，若存在则返回错误码，反之返回0表示`*logical`给定的逻辑块起始地址是文件之中地址最小的逻辑块并且设置返回的逻辑块为0。

当`ex`给定的`extent`不是B+树中第一个`extent`时，这个`extent`之中保存的逻辑块之中最后一个块就是在`*logical`给定的逻辑块起始地址之前并且最靠近它的已分配的逻辑块(单个块)。

### `ext4_ext_search_right`函数

```c
/*
 * Search the closest allocated block to the right for *logical
 * and returns it at @logical + it's physical address at @phys.
 * If not exists, return 0 and @phys is set to 0. We will return
 * 1 which means we found an allocated block and ret_ex is valid.
 * Or return a (< 0) error code.
 */
static int ext4_ext_search_right(struct inode *inode,
				 struct ext4_ext_path *path,
				 ext4_lblk_t *logical, ext4_fsblk_t *phys,
				 struct ext4_extent *ret_ex)
{
	struct buffer_head *bh = NULL;
	struct ext4_extent_header *eh;
	struct ext4_extent_idx *ix;
	struct ext4_extent *ex;
	int depth;	/* Note, NOT eh_depth; depth from top of tree */
	int ee_len;

	if (unlikely(path == NULL)) {
		EXT4_ERROR_INODE(inode, "path == NULL *logical %d!", *logical);
		return -EFSCORRUPTED;
	}
	depth = path->p_depth;
	*phys = 0;

	if (depth == 0 && path->p_ext == NULL)
		return 0;

	/* usually extent in the path covers blocks smaller
	 * then *logical, but it can be that extent is the
	 * first one in the file */

	ex = path[depth].p_ext;
	ee_len = ext4_ext_get_actual_len(ex);
	if (*logical < le32_to_cpu(ex->ee_block)) {
		if (unlikely(EXT_FIRST_EXTENT(path[depth].p_hdr) != ex)) {
			EXT4_ERROR_INODE(inode,
					 "first_extent(path[%d].p_hdr) != ex",
					 depth);
			return -EFSCORRUPTED;
		}
		while (--depth >= 0) {
			ix = path[depth].p_idx;
			if (unlikely(ix != EXT_FIRST_INDEX(path[depth].p_hdr))) {
				EXT4_ERROR_INODE(inode,
						 "ix != EXT_FIRST_INDEX *logical %d!",
						 *logical);
				return -EFSCORRUPTED;
			}
		}
		goto found_extent;
	}

	if (unlikely(*logical < (le32_to_cpu(ex->ee_block) + ee_len))) {
		EXT4_ERROR_INODE(inode,
				 "logical %d < ee_block %d + ee_len %d!",
				 *logical, le32_to_cpu(ex->ee_block), ee_len);
		return -EFSCORRUPTED;
	}

	if (ex != EXT_LAST_EXTENT(path[depth].p_hdr)) {
		/* next allocated block in this leaf */
		ex++;
		goto found_extent;
	}

	/* go up and search for index to the right */
	while (--depth >= 0) {
		ix = path[depth].p_idx;
		if (ix != EXT_LAST_INDEX(path[depth].p_hdr))
			goto got_index;
	}

	/* we've gone up to the root and found no index to the right */
	return 0;

got_index:
	/* we've found index to the right, let's
	 * follow it and find the closest allocated
	 * block to the right */
	ix++;
	while (++depth < path->p_depth) {
		/* subtract from p_depth to get proper eh_depth */
		bh = read_extent_tree_block(inode, ix, path->p_depth - depth, 0);
		if (IS_ERR(bh))
			return PTR_ERR(bh);
		eh = ext_block_hdr(bh);
		ix = EXT_FIRST_INDEX(eh);
		put_bh(bh);
	}

	bh = read_extent_tree_block(inode, ix, path->p_depth - depth, 0);
	if (IS_ERR(bh))
		return PTR_ERR(bh);
	eh = ext_block_hdr(bh);
	ex = EXT_FIRST_EXTENT(eh);
found_extent:
	*logical = le32_to_cpu(ex->ee_block);
	*phys = ext4_ext_pblock(ex);
	if (ret_ex)
		*ret_ex = *ex;
	if (bh)
		put_bh(bh);
	return 1;
}
```

这个函数搜索在`*logic`给定逻辑块右侧并且距离最近的已分配逻辑块(单个块)以及这个逻辑块(单个块)对应的物理块，搜索到的逻辑块(单个块)通过`logical`参数返回、物理块通过`phys`参数返回、搜索到的逻辑块(单个块)所在的`extent`通过`ret_ex`返回，这个函数的基本参数检查与`ext4_ext_search_left`函数的基本参数检查内容一致。

当`ex`给定的`extent`对应文件之中第一个`extent`时处理逻辑与`ext4_ext_search_left`函数对应的处理逻辑相同，只不过这个函数是在搜索`*logical`给定逻辑块右侧的已经分配的逻辑块，这个`extent`之中保存的逻辑块起始地址正好是要搜索的已分配的逻辑块起始地址，跳转到标签`found_extent`处继续执行。这个函数之中文件中第一个`extent`的判断逻辑以及检查逻辑见`ext4_ext_search_right`函数对应的流程记录。

当`ex`给定的`extent`不是其所在节点的最后一个`extent`时，它后边的`extent`之中保存的逻辑块起始地址即为搜索到的已经分配的逻辑块(单个块)地址，跳转到标签`found_extent`处继续执行；当`ex`给定的`extent`是所在节点的最后一个`extent`时，搜索`path`之中上层中的索引，如找到一个索引不是所在节点的最后一个索引就跳转到标签`got_index`处继续执行，若`path`之中上层的所有索引都是所在节点的最后一个索引那么返回0表明为找到这样的逻辑块。

标签`got_index`处的逻辑为从找到的索引所在的节点之中位于下一个位置的索引开始，沿着B+树向下层遍历直到找到叶子节点，把它当作待读取节点的索引，遍历过程中根据待读取节点的索引从磁盘中加载这层的索引节点、获取到节点之中第一个索引，把这个索引当作遍历下一层时使用的待读取节点的索引。找到叶子节点之后叶子节点之中的第一个`extent`之中保存的逻辑起始地址即为搜索到的已分配的逻辑块(单个块)的地址，将这个`extent`保存在`ex`之中。

标签`found_extent`处的逻辑比较简单，`ex`给定了搜索到的逻辑块(单个块)所在的`extent`，那么更新`*logical`为这个`extent`之中保存的逻辑块起始地址，`*phys`之中保存逻辑地址对应的物理地址，返回1表示已经找到。

### `ext4_ext_check_overlap`函数

```c
/*
 * check if a portion of the "newext" extent overlaps with an
 * existing extent.
 *
 * If there is an overlap discovered, it updates the length of the newext
 * such that there will be no overlap, and then returns 1.
 * If there is no overlap found, it returns 0.
 */
static unsigned int ext4_ext_check_overlap(struct ext4_sb_info *sbi,
					   struct inode *inode,
					   struct ext4_extent *newext,
					   struct ext4_ext_path *path)
{
	ext4_lblk_t b1, b2;
	unsigned int depth, len1;
	unsigned int ret = 0;

	b1 = le32_to_cpu(newext->ee_block);
	len1 = ext4_ext_get_actual_len(newext);
	depth = ext_depth(inode);
	if (!path[depth].p_ext)
		goto out;
	b2 = EXT4_LBLK_CMASK(sbi, le32_to_cpu(path[depth].p_ext->ee_block));

	/*
	 * get the next allocated block if the extent in the path
	 * is before the requested block(s)
	 */
	if (b2 < b1) {
		b2 = ext4_ext_next_allocated_block(path);
		if (b2 == EXT_MAX_BLOCKS)
			goto out;
		b2 = EXT4_LBLK_CMASK(sbi, b2);
	}

	/* check for wrap through zero on extent logical start block*/
	if (b1 + len1 < b1) {
		len1 = EXT_MAX_BLOCKS - b1;
		newext->ee_len = cpu_to_le16(len1);
		ret = 1;
	}

	/* check for overlap */
	if (b1 + len1 > b2) {
		newext->ee_len = cpu_to_le16(b2 - b1);
		ret = 1;
	}
out:
	return ret;
}
```

这个函数查找`newex`给定的逻辑块是否与一个已经创建的`extent`重叠，未发现重叠返回1，发现重叠时修改`newex`之中逻辑块长度使得逻辑块与已经创建的`extent`不再有重叠。函数中`b1`为`newex`指定的逻辑块起始地址、`len1`为逻辑块长度、`b2`为搜索到的`extent`之中保存的逻辑块所在`cluster`中的第一个逻辑块(单个块)的地址，这个函数的主要流程如下：

1).若`newex`给定的逻辑块在搜索到的`extent`之前，通过`ext4_ext_next_allocated_block`获取到下一个`extent`之中保存的逻辑块起始地址并保存到`b2`之中，如果无法获取到则跳转到`out`向调用者返回0，获取到更新`b2`为所在`cluster`中的起始逻辑块(单个块)地址。只有在`newex`给定的逻辑块起始地址之后的`extent`与给定逻辑块之间才能出现可以分配的逻辑块，所以这种出现这种情况需要获取到搜索到的`extent`所在节点中下一个位置中的`extent`；

2).若`newex`给定逻辑块结束地址计算过程中出现数值溢出，需要对逻辑块的长度进行调整，将逻辑块的长度调整为`ext4`文件系统在给定的逻辑块起始地址之后还能够容纳的最大的逻辑块数量，调整完成之后设置返回值为1；

3).若`newex`给定的逻辑块的结束地址大于之后的`extent`所在`cluster`中的起始逻辑块(单个块)的地址，意味着给定的逻辑块与之后的`extent`所在的`cluster`有重叠，调整`newex`之中给定逻辑块的长度为给定逻辑块起始地址到`cluster`之中第一个逻辑块(单个块)起始地址之间可分配逻辑块个数，设置返回值为1。这里需要注意的是`b2`为`cluster`中起始逻辑块(单个块)的地址，这意味着调整之后`newex`给定逻辑块不能进入`b2`所属的`cluster`之中，这样是为了避免在后续流程分配`cluster`时占用的逻辑块与`b2`所属`cluster`发生冲突；

4).标签`out`处的代码返回指定的返回值。

### `ext4_ext_find_goal`函数

```c
static ext4_fsblk_t ext4_ext_find_goal(struct inode *inode,
			      struct ext4_ext_path *path,
			      ext4_lblk_t block)
{
	if (path) {
		int depth = path->p_depth;
		struct ext4_extent *ex;

		/*
		 * Try to predict block placement assuming that we are
		 * filling in a file which will eventually be
		 * non-sparse --- i.e., in the case of libbfd writing
		 * an ELF object sections out-of-order but in a way
		 * the eventually results in a contiguous object or
		 * executable file, or some database extending a table
		 * space file.  However, this is actually somewhat
		 * non-ideal if we are writing a sparse file such as
		 * qemu or KVM writing a raw image file that is going
		 * to stay fairly sparse, since it will end up
		 * fragmenting the file system's free space.  Maybe we
		 * should have some hueristics or some way to allow
		 * userspace to pass a hint to file system,
		 * especially if the latter case turns out to be
		 * common.
		 */
		ex = path[depth].p_ext;
		if (ex) {
			ext4_fsblk_t ext_pblk = ext4_ext_pblock(ex);
			ext4_lblk_t ext_block = le32_to_cpu(ex->ee_block);

			if (block > ext_block)
				return ext_pblk + (block - ext_block);
			else
				return ext_pblk - (ext_block - block);
		}

		/* it looks like index is empty;
		 * try to find starting block from index itself */
		if (path[depth].p_bh)
			return path[depth].p_bh->b_blocknr;
	}

	/* OK. use inode's group */
	return ext4_inode_to_goal_block(inode);
}
```

这个函数用于确定接下来分配`block`给定逻辑块对应物理块过程中搜索物理块使用的起始地址，主要分为以下几种情况：

1).若之前的搜索过程中找到了一个`extent`，根据`block`给定的逻辑块起始地址与`extent`之中保存的逻辑块起始地址相对大小计算物理块起始地址并返回，代码之中`ext_pblk`为`extent`之中保存的物理块起始地址、`ext_block`为`extent`之中保存的逻辑块起始地址，两个逻辑块起始地址之间的差值也就是两个物理块起始地址之间的差值，通过这个差值可以由`extent`之中保存的物理块起始地址推测出`block`对应的物理块起始地址；

2).若遍历过程中遍历到了索引但是没有找到对应的`extent`，返回这个索引所在的物理块(单个块)地址，在搜索某个逻辑块起始地址附近的`extent`时若出现逻辑块起始地址超出了当前`extent`映射的范围会出现遍历过程中找到了索引但是没有找到对应`extent`的情况；

3).其他的情况下调用`ext4_inode_to_goal_block`函数使用inode所在的块组之中寻找一个物理块，例如当文件为空文件的时候`path`为空；

### `ext4_inode_to_goal_block`函数

```c
/**
 *	ext4_inode_to_goal_block - return a hint for block allocation
 *	@inode: inode for block allocation
 *
 *	Return the ideal location to start allocating blocks for a
 *	newly created inode.
 */
ext4_fsblk_t ext4_inode_to_goal_block(struct inode *inode)
{
	struct ext4_inode_info *ei = EXT4_I(inode);
	ext4_group_t block_group;
	ext4_grpblk_t colour;
	int flex_size = ext4_flex_bg_size(EXT4_SB(inode->i_sb));
	ext4_fsblk_t bg_start;
	ext4_fsblk_t last_block;

	block_group = ei->i_block_group;
	if (flex_size >= EXT4_FLEX_SIZE_DIR_ALLOC_SCHEME) {
		/*
		 * If there are at least EXT4_FLEX_SIZE_DIR_ALLOC_SCHEME
		 * block groups per flexgroup, reserve the first block
		 * group for directories and special files.  Regular
		 * files will start at the second block group.  This
		 * tends to speed up directory access and improves
		 * fsck times.
		 */
		block_group &= ~(flex_size-1);
		if (S_ISREG(inode->i_mode))
			block_group++;
	}
	bg_start = ext4_group_first_block_no(inode->i_sb, block_group);
	last_block = ext4_blocks_count(EXT4_SB(inode->i_sb)->s_es) - 1;

	/*
	 * If we are doing delayed allocation, we don't need take
	 * colour into account.
	 */
	if (test_opt(inode->i_sb, DELALLOC))
		return bg_start;

	if (bg_start + EXT4_BLOCKS_PER_GROUP(inode->i_sb) <= last_block)
		colour = (task_pid_nr(current) % 16) *
			(EXT4_BLOCKS_PER_GROUP(inode->i_sb) / 16);
	else
		colour = (task_pid_nr(current) % 16) *
			((last_block - bg_start) / 16);
	return bg_start + colour;
}
```

这个函数返回inode所在块组或者下一个块组中的一个物理块地址当作物理块分配时起始地址，`block_group`为inode所在的块组、`bg_start`为这个块组起始物理块(单个块)地址、`last_block`为文件系统中最后一个物理块(单个块)的地址，分区的大小会影响`last_block`的值。函数的主要流程如下：

1).在`ext4`文件系统中`flex group`由多个块组组成，若`flex group`之中块的数量大于`EXT4_FLEX_SIZE_DIR_ALLOC_SCHEME`，inode所在的块组用于存储目录和特殊的文件、下一个块组用于分配普通文件，代码之中使用`block_group &= ~(flex_size-1)`将`block_group`设置为`flex group`的起始块组、通过`S_ISREG(inode->i_mode)`判断是否在分配普通文件使用的物理块，调整`block group`之后需要调整`bg_start`为`block group`给定的块组中起始物理块地址(单个块)；

2).若正在进行的物理块分配为延迟分配，直接返回`block group`给定的块组中起始物理块(单个块)地址；

3).若块组中结束物理块(单个块)没有超过文件系统的限制，返回的物理块(单个块)地址在快组中的偏移计算结合任务id以及块组之中物理块的个数；

4)若块组中结束物理块(单个块)超过文件系统的限制，返回的物理块(单个块)地址在物理块中的偏移计算结合任务id以及块组中的可用物理块数量；

### `ext4_mb_new_blocks`函数

```c
/*
 * Main entry point into mballoc to allocate blocks
 * it tries to use preallocation first, then falls back
 * to usual allocation
 */
ext4_fsblk_t ext4_mb_new_blocks(handle_t *handle,
				struct ext4_allocation_request *ar, int *errp)
{
	struct ext4_allocation_context *ac = NULL;
	struct ext4_sb_info *sbi;
	struct super_block *sb;
	ext4_fsblk_t block = 0;
	unsigned int inquota = 0;
	unsigned int reserv_clstrs = 0;
	int retries = 0;
	u64 seq;

	might_sleep();
	sb = ar->inode->i_sb;
	sbi = EXT4_SB(sb);

	trace_ext4_request_blocks(ar);
	if (sbi->s_mount_state & EXT4_FC_REPLAY)
		return ext4_mb_new_blocks_simple(ar, errp);

	/* Allow to use superuser reservation for quota file */
	if (ext4_is_quota_file(ar->inode))
		ar->flags |= EXT4_MB_USE_ROOT_BLOCKS;

	if ((ar->flags & EXT4_MB_DELALLOC_RESERVED) == 0) {
		/* Without delayed allocation we need to verify
		 * there is enough free blocks to do block allocation
		 * and verify allocation doesn't exceed the quota limits.
		 */
		while (ar->len &&
			ext4_claim_free_clusters(sbi, ar->len, ar->flags)) {

			/* let others to free the space */
			cond_resched();
			ar->len = ar->len >> 1;
		}
		if (!ar->len) {
			ext4_mb_show_pa(sb);
			*errp = -ENOSPC;
			return 0;
		}
		reserv_clstrs = ar->len;
		if (ar->flags & EXT4_MB_USE_ROOT_BLOCKS) {
			dquot_alloc_block_nofail(ar->inode,
						 EXT4_C2B(sbi, ar->len));
		} else {
			while (ar->len &&
				dquot_alloc_block(ar->inode,
						  EXT4_C2B(sbi, ar->len))) {

				ar->flags |= EXT4_MB_HINT_NOPREALLOC;
				ar->len--;
			}
		}
		inquota = ar->len;
		if (ar->len == 0) {
			*errp = -EDQUOT;
			goto out;
		}
	}

	ac = kmem_cache_zalloc(ext4_ac_cachep, GFP_NOFS);
	if (!ac) {
		ar->len = 0;
		*errp = -ENOMEM;
		goto out;
	}

	*errp = ext4_mb_initialize_context(ac, ar);
	if (*errp) {
		ar->len = 0;
		goto out;
	}

	ac->ac_op = EXT4_MB_HISTORY_PREALLOC;
	seq = this_cpu_read(discard_pa_seq);
	if (!ext4_mb_use_preallocated(ac)) {
		ac->ac_op = EXT4_MB_HISTORY_ALLOC;
		ext4_mb_normalize_request(ac, ar);

		*errp = ext4_mb_pa_alloc(ac);
		if (*errp)
			goto errout;
repeat:
		/* allocate space in core */
		*errp = ext4_mb_regular_allocator(ac);
		/*
		 * pa allocated above is added to grp->bb_prealloc_list only
		 * when we were able to allocate some block i.e. when
		 * ac->ac_status == AC_STATUS_FOUND.
		 * And error from above mean ac->ac_status != AC_STATUS_FOUND
		 * So we have to free this pa here itself.
		 */
		if (*errp) {
			ext4_mb_pa_free(ac);
			ext4_discard_allocated_blocks(ac);
			goto errout;
		}
		if (ac->ac_status == AC_STATUS_FOUND &&
			ac->ac_o_ex.fe_len >= ac->ac_f_ex.fe_len)
			ext4_mb_pa_free(ac);
	}
	if (likely(ac->ac_status == AC_STATUS_FOUND)) {
		*errp = ext4_mb_mark_diskspace_used(ac, handle, reserv_clstrs);
		if (*errp) {
			ext4_discard_allocated_blocks(ac);
			goto errout;
		} else {
			block = ext4_grp_offs_to_block(sb, &ac->ac_b_ex);
			ar->len = ac->ac_b_ex.fe_len;
		}
	} else {
		if (++retries < 3 &&
		    ext4_mb_discard_preallocations_should_retry(sb, ac, &seq))
			goto repeat;
		/*
		 * If block allocation fails then the pa allocated above
		 * needs to be freed here itself.
		 */
		ext4_mb_pa_free(ac);
		*errp = -ENOSPC;
	}

errout:
	if (*errp) {
		ac->ac_b_ex.fe_len = 0;
		ar->len = 0;
		ext4_mb_show_ac(ac);
	}
	ext4_mb_release_context(ac);
out:
	if (ac)
		kmem_cache_free(ext4_ac_cachep, ac);
	if (inquota && ar->len < inquota)
		dquot_free_block(ar->inode, EXT4_C2B(sbi, inquota - ar->len));
	if (!ar->len) {
		if ((ar->flags & EXT4_MB_DELALLOC_RESERVED) == 0)
			/* release all the reserved blocks if non delalloc */
			percpu_counter_sub(&sbi->s_dirtyclusters_counter,
						reserv_clstrs);
	}

	trace_ext4_allocate_blocks(ar, (unsigned long long)block);

	return block;
}
```

#### 日志重放

当系统异常崩溃时，未提交的事务会被记录在日志之中，当重新挂载文件系统时日志重放机制会恢复事务，确保文件系统恢复到一致状态，这部分内容详细参见[Journal(jbd2)](https://www.kernel.org/doc/html/latest/filesystems/ext4/journal.html)之中的内容。当超级块设置了`EXT4_FC_REPLAY`标志时意味着此时在进行日志重放，调用`ext4_mb_new_blocks_simple`函数分配多个逻辑块，这个函数从`ar`给定的物理块(单个块)起始地址开始线性搜索可用的物理块，在搜索过程中排除在日志重放之后即将使用的块。

#### 配额检查

若现在分配的物理块用于存储配额文件的内容，允许使用为超级用于预留的物理块，使用`ext4_is_quota_file`函数检查分配的逻辑块是否用于配额文件内容存储，添加`EXT4_MB_USE_ROOT_BLOCKS`标记意味着在之后的分配流程中允许使用为超级用户预留的物理块。

若分配过程中关闭预分配机制，需要保证目前文件系统中有足够的`cluster`满足分配需求(即`ar`给定的物理块分配过程中的条件，例如`ar->len`给定的待分配的`cluster`数量、从`ar->goal`开始搜索可用的物理块等)并且分配需求没有超出配额的限制。代码之中使用一个`while`循环判断文件系统中可用的`cluster`是否能够满足分配需求，若不满足将分配需求减半继续进行循环、满足分配需求或者分配需求降成0(即`ar->len`成为0)退出循环。若无法满足分配需求返回0并设置对应的错误码，若能够满足分配需求进行配额检查：若允许使用为超级用户预留的物理块时从配额之中加上待分配的物理块数量；其他情况使用`while`循环判断分配需求能否满足配额限制，若不满足降低分配需求重新进行循环、若配额之后加上待分配的物理块数量并且满足配额限制退出循环、分配需求降成0退出循环，当因为分配需求降成0退出循环时返回0并设置对应的错误码。配额检查过程之后会将`inquota`设置为待分配的`cluster`数量，在后续的流程中会与实际分配到的物理块进行对比。

#### `cluster`分配

此部分代码流程如下：
1).创建物理块分配上下文然后进行初始化，初始化过程在对待分配物理块数量超过块组中物理块数量时对待分配物理块数量进行修正、计算搜索起始物理块所在的块组以及块组内的偏移、确定分配策略(使用块组分配还是使用流式分配，流式分配尝试尽量为文件分配连续的物理块)；

2).若分配过程中无法使用预分配的物理块，调用`ext4_mb_normalize_request`函数对待分配的物理块起始地址和物理块数量进行规范化以满足地址对齐、预分配等要求，这个函数确定并填充待分配的物理块所在块组以及块组内`cluster`的偏移至`ac->ac_g_ex.fe_group`和`ac->ac_g_ex.fe_start`之中，创建用于预分配机制的`struct ext4_prealloc_space`实例、尝试按照分配请求`ac`之中给定的物理块起始地址和物理块数量分配物理块(对应`ext4_mb_regular_allocator`函数调用，这个函数调用的位置也是标签`repeat`所在位置)，若分配过程中出现错误，释放为预分配机制创建的实例、释放分配过程中已经分配的物理块，跳转到`errout`处继续执行；

3).当分配过程中可以使用预分配的物理块或者分配物理块过程中没有出现错误，需要考虑分配过程中是否找到了符合分配需求的物理块：若找到了已经满足分配需求的物理块，释放为预分配机制创建的实例，标记已分配的物理块为已使用状态、更新组描述符、全局计数器等元数据，获取分配得到物理块的起始地址以及分配得到的物理块数量；若未找到满足分配需求的物理块，最多进行三次物理块分配，每次重试前先释放预分配的物理块并且判断是否需要继续进行重试，若可以进行重试跳转到标签`repeat`处执行。若重试三次之后也无法找到满足分配需求的物理块，释放为预分配机制创建的实例，设置错误码；

这部分代码涉及到了许多关键函数，`ext4_mb_use_preallocated`函数用于确定是否可以使用预分配的物理块、`ext4_mb_normalize_request`用于对分配请求进行规范化、`ext4_mb_regular_allocator`函数用于进行物理块分配、`ext4_mb_mark_diskspace_used`用于标记已经分配的物理块，在后边记录这些函数的流程。

#### 标签`errout`和`out`

此部分代码释放分配过程中创建的上下文，若实际分配的`cluster`数量小于`inquota`之中保存的数量需要从配额之中减去少分配的`cluster`对应的物理块数量，返回分配到的物理块起始地址。

### `ext4_mb_regular_allocator` 函数

```c
static noinline_for_stack int
ext4_mb_regular_allocator(struct ext4_allocation_context *ac)
{
	ext4_group_t prefetch_grp = 0, ngroups, group, i;
	int cr = -1, new_cr;
	int err = 0, first_err = 0;
	unsigned int nr = 0, prefetch_ios = 0;
	struct ext4_sb_info *sbi;
	struct super_block *sb;
	struct ext4_buddy e4b;
	int lost;

	sb = ac->ac_sb;
	sbi = EXT4_SB(sb);
	ngroups = ext4_get_groups_count(sb);
	/* non-extent files are limited to low blocks/groups */
	if (!(ext4_test_inode_flag(ac->ac_inode, EXT4_INODE_EXTENTS)))
		ngroups = sbi->s_blockfile_groups;

	BUG_ON(ac->ac_status == AC_STATUS_FOUND);

	/* first, try the goal */
	err = ext4_mb_find_by_goal(ac, &e4b);
	if (err || ac->ac_status == AC_STATUS_FOUND)
		goto out;

	if (unlikely(ac->ac_flags & EXT4_MB_HINT_GOAL_ONLY))
		goto out;

	/*
	 * ac->ac_2order is set only if the fe_len is a power of 2
	 * if ac->ac_2order is set we also set criteria to 0 so that we
	 * try exact allocation using buddy.
	 */
	i = fls(ac->ac_g_ex.fe_len);
	ac->ac_2order = 0;
	/*
	 * We search using buddy data only if the order of the request
	 * is greater than equal to the sbi_s_mb_order2_reqs
	 * You can tune it via /sys/fs/ext4/<partition>/mb_order2_req
	 * We also support searching for power-of-two requests only for
	 * requests upto maximum buddy size we have constructed.
	 */
	if (i >= sbi->s_mb_order2_reqs && i <= MB_NUM_ORDERS(sb)) {
		/*
		 * This should tell if fe_len is exactly power of 2
		 */
		if ((ac->ac_g_ex.fe_len & (~(1 << (i - 1)))) == 0)
			ac->ac_2order = array_index_nospec(i - 1,
							   MB_NUM_ORDERS(sb));
	}

	/* if stream allocation is enabled, use global goal */
	if (ac->ac_flags & EXT4_MB_STREAM_ALLOC) {
		/* TBD: may be hot point */
		spin_lock(&sbi->s_md_lock);
		ac->ac_g_ex.fe_group = sbi->s_mb_last_group;
		ac->ac_g_ex.fe_start = sbi->s_mb_last_start;
		spin_unlock(&sbi->s_md_lock);
	}

	/* Let's just scan groups to find more-less suitable blocks */
	cr = ac->ac_2order ? 0 : 1;
	/*
	 * cr == 0 try to get exact allocation,
	 * cr == 3  try to get anything
	 */
repeat:
	for (; cr < 4 && ac->ac_status == AC_STATUS_CONTINUE; cr++) {
		ac->ac_criteria = cr;
		/*
		 * searching for the right group start
		 * from the goal value specified
		 */
		group = ac->ac_g_ex.fe_group;
		ac->ac_groups_linear_remaining = sbi->s_mb_max_linear_groups;
		prefetch_grp = group;

		for (i = 0, new_cr = cr; i < ngroups; i++,
		     ext4_mb_choose_next_group(ac, &new_cr, &group, ngroups)) {
			int ret = 0;

			cond_resched();
			if (new_cr != cr) {
				cr = new_cr;
				goto repeat;
			}

			/*
			 * Batch reads of the block allocation bitmaps
			 * to get multiple READs in flight; limit
			 * prefetching at cr=0/1, otherwise mballoc can
			 * spend a lot of time loading imperfect groups
			 */
			if ((prefetch_grp == group) &&
			    (cr > 1 ||
			     prefetch_ios < sbi->s_mb_prefetch_limit)) {
				unsigned int curr_ios = prefetch_ios;

				nr = sbi->s_mb_prefetch;
				if (ext4_has_feature_flex_bg(sb)) {
					nr = 1 << sbi->s_log_groups_per_flex;
					nr -= group & (nr - 1);
					nr = min(nr, sbi->s_mb_prefetch);
				}
				prefetch_grp = ext4_mb_prefetch(sb, group,
							nr, &prefetch_ios);
				if (prefetch_ios == curr_ios)
					nr = 0;
			}

			/* This now checks without needing the buddy page */
			ret = ext4_mb_good_group_nolock(ac, group, cr);
			if (ret <= 0) {
				if (!first_err)
					first_err = ret;
				continue;
			}

			err = ext4_mb_load_buddy(sb, group, &e4b);
			if (err)
				goto out;

			ext4_lock_group(sb, group);

			/*
			 * We need to check again after locking the
			 * block group
			 */
			ret = ext4_mb_good_group(ac, group, cr);
			if (ret == 0) {
				ext4_unlock_group(sb, group);
				ext4_mb_unload_buddy(&e4b);
				continue;
			}

			ac->ac_groups_scanned++;
			if (cr == 0)
				ext4_mb_simple_scan_group(ac, &e4b);
			else if (cr == 1 && sbi->s_stripe &&
					!(ac->ac_g_ex.fe_len % sbi->s_stripe))
				ext4_mb_scan_aligned(ac, &e4b);
			else
				ext4_mb_complex_scan_group(ac, &e4b);

			ext4_unlock_group(sb, group);
			ext4_mb_unload_buddy(&e4b);

			if (ac->ac_status != AC_STATUS_CONTINUE)
				break;
		}
		/* Processed all groups and haven't found blocks */
		if (sbi->s_mb_stats && i == ngroups)
			atomic64_inc(&sbi->s_bal_cX_failed[cr]);
	}

	if (ac->ac_b_ex.fe_len > 0 && ac->ac_status != AC_STATUS_FOUND &&
	    !(ac->ac_flags & EXT4_MB_HINT_FIRST)) {
		/*
		 * We've been searching too long. Let's try to allocate
		 * the best chunk we've found so far
		 */
		ext4_mb_try_best_found(ac, &e4b);
		if (ac->ac_status != AC_STATUS_FOUND) {
			/*
			 * Someone more lucky has already allocated it.
			 * The only thing we can do is just take first
			 * found block(s)
			 */
			lost = atomic_inc_return(&sbi->s_mb_lost_chunks);
			mb_debug(sb, "lost chunk, group: %u, start: %d, len: %d, lost: %d\n",
				 ac->ac_b_ex.fe_group, ac->ac_b_ex.fe_start,
				 ac->ac_b_ex.fe_len, lost);

			ac->ac_b_ex.fe_group = 0;
			ac->ac_b_ex.fe_start = 0;
			ac->ac_b_ex.fe_len = 0;
			ac->ac_status = AC_STATUS_CONTINUE;
			ac->ac_flags |= EXT4_MB_HINT_FIRST;
			cr = 3;
			goto repeat;
		}
	}

	if (sbi->s_mb_stats && ac->ac_status == AC_STATUS_FOUND)
		atomic64_inc(&sbi->s_bal_cX_hits[ac->ac_criteria]);
out:
	if (!err && ac->ac_status != AC_STATUS_FOUND && first_err)
		err = first_err;

	mb_debug(sb, "Best len %d, origin len %d, ac_status %u, ac_flags 0x%x, cr %d ret %d\n",
		 ac->ac_b_ex.fe_len, ac->ac_o_ex.fe_len, ac->ac_status,
		 ac->ac_flags, cr, err);

	if (nr)
		ext4_mb_prefetch_fini(sb, prefetch_grp, nr);

	return err;
}
```

这个函数是`ext4`文件系统中物理块分配的入口，物理块分配所需的信息在之前的流程中都已经存放到分配上下文`ac`之中了，函数的流程如下：

#### 从指定位置分配

尝试从指定的物理块(单个块)开始搜索物理块物理块，对应的函数为`ext4_mb_find_by_goal`，这个函数找到了符合要求的物理块意味着分配成功。

#### 设置伙伴算法参数

如果待分配物理块数量为2的幂并且幂值在指定的范围之内，设置`ac->ac_2order`为这个幂值，幂值的范围由两个值确定：`s_mb_order2_reqs`是启动伙伴算法的阈值，`MB_NUM_ORDERS(sb)`返回伙伴算法能够支持的最大幂值。代码中使用的`array_index_nospec`与现代cpu的推测执行机制有关，在推测执行期间可能会跳过条件判断直接进行数组访问，此时推测得到的数组下表可能会超过数组边界，需要通过生成掩码来限制数组下标不超过数组边界，这里使用`array_index_nospec`来限制在推测执行时`i-1`的值不超过`MB_NUM_ORDERS(sb)`。

#### 设置流式分配参数

如果使用流式分配，设置物理块搜索的起始地址，起始地址由上一次流式得到的物理块所在块组以及组内偏移组成，`sbi->s_mb_last_group`为上一次流式分配的物理块所在块组、`sbi->s_mb_last_start`为这个物理块在快组内的偏移。

#### 空闲物理块搜索

`ext4`文件系统中提供了三种不同的可用物理块查找策略，扫描策略由简单到复杂分别是`ext4_mb_simple_scan_group`对应的简单扫描策略、`ext4_mb_scan_aligned`对应的对其扫描策略、`ext4_mb_complex_scan_group`对应的复杂扫描策略，可用物理块查找过程中会先尝试使用简单扫描策略查找，若没有找到满足分配需求的物理块则使用更复杂的扫描策略。无论使用那种扫描策略，标记扫描到的空闲物理块为已使用状态，保存这个物理块的位置和长度，完成部分或者全部的物理块分配。空闲物理块搜索标签`repeat`到标签`out`之间的代码，这部分代码由一个两层`for`循环以及循环结束之后的处理组成，外层`for`循环确定使用哪个扫描策略、内层`for`循环尝试在所有的块组之中使用外层循环指定的搜索策略扫描空闲的物理块，循环之后的代码涉及到搜索了比较长的时间但未找到合适的物理块这种情况的处理。

内层`for`循环的主要逻辑如下：

1).寻找用于扫描空闲物理块的块组，对应的函数为`ext4_mb_choose_next_group`，这个函数将找到的块组编号保存到`group`之中；

2).预取块分配位图：

2.1).当使用复杂的搜索策略(`cr>1`)、预取过程的IO操作次数没有超过阈值、上一步找到的块组与待预取的块组为同一个块组时会执行块组的块位图预取操作，内层循环开始前设置待预取的块组为指定物理块(单个块)所在的块组、内存循环过程中将待预取的块组设置为预取函数(`ext4_mb_prefetch`函数)返回的下一个待预取的块组；

2.2).调用`ext4_mb_prefetch`函数进行块组位图预取，`nr`为需要预取的块组位图数量，`nr`的值默认为超级块中设置的每次预取的块组位图数量，如果启用了`flex_bg`特性则需要对`nr`的值进行更新，更新过程中考虑`flex_bg`特性设置的块组位图预取数量、待预取的块组在`flex_bg`中的偏移、超级块中每次预取的块组数量，考虑待预取的块组在`flex_bg`中的偏移是因为要考虑待预取的块组在`flex group`中的位置，只需要预取这个位置之后的块组。`ext4_mb_prefetch`函数返回下一个待预取的块组，当预取过程中未发生IO操作时意味着没有预取到任何一个块组的位图，设置`nr`为0；

3).检查`group`给定的块组是否可以用于物理块分配，对应的函数是`ext4_mb_good_group`，调用这个函数的时候不能持有块组锁；

4).加载伙伴信息(包含伙伴位图等内容)、持有块组锁，对应的函数是`ext4_mb_load_buddy`以及`ext4_lock_group`，块组锁是为了防止多线程同时修改一个块组中的内容；

5).再次检查`group`给定的块组是否可以用于物理块分配，潜在的原因例如在上次检查和持有块组锁期间有其他的线程使用了这些块组；

6).使用`cr`指定的搜索策略从`group`给定的块组之中搜索可用的物理块，`cr`的值由外层循环修改，不同的`cr`值意味着使用不同的搜索策略；

7).释放块组锁以及伙伴信息，内层循环至此结束；

两层`for`循环结束之后，如果搜索物理块是否消耗了比较长的时间，就需要先分配那些已经扫描到的物理块(使用`ext4_mb_try_best_found`函数)，若分配之后依然没有满足分配需求则清空分配上下文`ac`之中扫描到的物理块信息、设置优先使用扫描到的第一个物理块(这个物理块一定是距离`goal`给定的物理块(单个块)最近的)标记、设置`cr`为3以表明使用复杂的搜索策略，跳转到标签`repeat`处继续执行，也即空闲物理块搜索代码逻辑开始的位置。搜索物理块消耗比较长时间成立的条件为已经找到了部分物理块但还未满足分配需求、没有设置优先使用扫描到的第一个物理块标记。

#### 返回分配结果

若在空闲物理块搜索过程中发生了块组位图的预取操作，调用`ext4_mb_prefetch_fini`函数进行块组位图预取的收尾工作(例如伙伴位图初始化等)。最后返回0意味着分配成功，其他的值意味着分配失败、返回值自身为对应的错误码。

### `ext4_mb_find_by_goal`函数

```c
static noinline_for_stack
int ext4_mb_find_by_goal(struct ext4_allocation_context *ac,
				struct ext4_buddy *e4b)
{
	ext4_group_t group = ac->ac_g_ex.fe_group;
	int max;
	int err;
	struct ext4_sb_info *sbi = EXT4_SB(ac->ac_sb);
	struct ext4_group_info *grp = ext4_get_group_info(ac->ac_sb, group);
	struct ext4_free_extent ex;

	if (!grp)
		return -EFSCORRUPTED;
	if (!(ac->ac_flags & (EXT4_MB_HINT_TRY_GOAL | EXT4_MB_HINT_GOAL_ONLY)))
		return 0;
	if (grp->bb_free == 0)
		return 0;

	err = ext4_mb_load_buddy(ac->ac_sb, group, e4b);
	if (err)
		return err;

	ext4_lock_group(ac->ac_sb, group);
	if (unlikely(EXT4_MB_GRP_BBITMAP_CORRUPT(e4b->bd_info)))
		goto out;

	max = mb_find_extent(e4b, ac->ac_g_ex.fe_start,
			     ac->ac_g_ex.fe_len, &ex);
	ex.fe_logical = 0xDEADFA11; /* debug value */

	if (max >= ac->ac_g_ex.fe_len && ac->ac_g_ex.fe_len == sbi->s_stripe) {
		ext4_fsblk_t start;

		start = ext4_grp_offs_to_block(ac->ac_sb, &ex);
		/* use do_div to get remainder (would be 64-bit modulo) */
		if (do_div(start, sbi->s_stripe) == 0) {
			ac->ac_found++;
			ac->ac_b_ex = ex;
			ext4_mb_use_best_found(ac, e4b);
		}
	} else if (max >= ac->ac_g_ex.fe_len) {
		BUG_ON(ex.fe_len <= 0);
		BUG_ON(ex.fe_group != ac->ac_g_ex.fe_group);
		BUG_ON(ex.fe_start != ac->ac_g_ex.fe_start);
		ac->ac_found++;
		ac->ac_b_ex = ex;
		ext4_mb_use_best_found(ac, e4b);
	} else if (max > 0 && (ac->ac_flags & EXT4_MB_HINT_MERGE)) {
		/* Sometimes, caller may want to merge even small
		 * number of blocks to an existing extent */
		BUG_ON(ex.fe_len <= 0);
		BUG_ON(ex.fe_group != ac->ac_g_ex.fe_group);
		BUG_ON(ex.fe_start != ac->ac_g_ex.fe_start);
		ac->ac_found++;
		ac->ac_b_ex = ex;
		ext4_mb_use_best_found(ac, e4b);
	}
out:
	ext4_unlock_group(ac->ac_sb, group);
	ext4_mb_unload_buddy(e4b);

	return 0;
}
```

这个函数从`ac`中给定的块组内某个`cluster`处开始寻找空闲物理块，使用`ext4_mb_load_buddy`加载块组的伙伴信息之后，使用`mb_find_extent`函数搜索给定块组内某个`cluster`附近的空闲物理块写入到`ex`之中并返回找到的空闲物理块长度，`mb_find_extent`函数找到的空闲物理块长度可能无法满足分配需求，需要对`mb_find_extent`函数的返回值进行判断，代码之中`max`保存的就是搜索到的空闲物理块长度。

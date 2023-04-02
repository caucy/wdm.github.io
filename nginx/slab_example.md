## 使用slab 实现多进程配置共享


### slab 核心代码

```

// 这里添加红黑树的初始化
ngx_rbtree_init(&conf->lcshm->rbtree, &conf->lcshm->sentinel,  \
rbtree_insert_value_func); // rbtree_insert_value_func 插入的回调函数

// 节点的插入
node->key = key;
node->data = 1;
ngx_rbtree_insert(&conf->lcshm->rbtree, node); // 会调用rbtree_insert_value_func 回调

// 节点的查询, 都是固定模版
// 参数r 方便日志的打印
ngx_int_t ngx_http_pagecount_lookup(ngx_http_request_t *r, ngx_http_location_conf_t *conf, ngx_uint_t key)
{
    ngx_rbtree_node_t *node, *sentinel;

	node = conf->lcshm->rbtree.root;
	sentinel = conf->lcshm->rbtree.sentinel;
	
	while(node != sentinel)   //  node == sentinel 需要进行下面操作，在slab中分配节点
	{
	    if(key < node->key)
	    {
	        node = node->left;
	        continue;
	    } 
	    else if (key > node->key)
	    {
	        node = node->right;
	        continue;
	    }
	    else
	    {
	        node->data ++;   // 找到了,  需要做一个原子操作
	        return NGX_OK;
	    }
	}
	
	// 添加之前 需要在slab中分配一个节点, 这其实是插入逻辑
	node = ngx_slab_alloc_locked(conf->pool, sizeof(ngx_rbtree_node_t));
	if (NULL == node) {
		return NGX_ERROR;
	}
	
	node->key = key;
	node->data = 1;
	ngx_rbtree_insert(&conf->lcshm->rbtree, node);
	
	return NGX_OK;

}

    
// 遍历节点
ngx_rbtree_node_t *node = ngx_rbtree_min(conf->lcshm->rbtree.root, \
conf->lcshm->rbtree.sentinel);

do {
	// 提取node 信息
	// ...


	node = ngx_rbtree_next(&conf->lcshm->rbtree, node);

} while (node);
```


# nginx 操作多进程共享内存

## 0. nginx 共享内存

### 1. 共享内存的实现方式
nginx 共享内存主要三种方式

* mmap anon 
* mmap /dev/zero
* shmctl 

nginx 通过宏判断到底选哪种实现，一般linux 选第一种方式, 具体可以参考[ngx_shm_alloc](https://github.com/nginx/nginx/blob/a64190933e06758d50eea926e6a55974645096fd/src/os/unix/ngx_shmem.c)



### 2. 共享内存的api



## 1. 使用atomic 

### atomic 的api
从语义上主要是 set、get、add、cas

```
#define ngx_atomic_cmp_set(lock, old, set)        __sync_bool_compare_and_swap(lock, old, set)
#define ngx_atomic_fetch_add(value, add)          __sync_fetch_and_add(value, add)
#define ngx_memory_barrier()                      __sync_synchronize() 
#define ngx_cpu_pause()                           __asm__("pause") 
```

### event.c 中有对全局连接数的一个很好的封装demo

```
// 1， 连接初始化
ngx_atomic_t   ngx_stat_accepted0;
ngx_atomic_t  *ngx_stat_accepted = &ngx_stat_accepted0;
ngx_atomic_t   ngx_stat_handled0;
ngx_atomic_t  *ngx_stat_handled = &ngx_stat_handled0;

// 2，共享内存的申请

typedef struct {
    u_char      *addr;
    size_t       size;
    ...
} ngx_shm_t;
ngx_int_t ngx_shm_alloc(ngx_shm_t *shm);
void ngx_shm_free(ngx_shm_t *shm);

// 3，连接的操作

shm.size = size;
ngx_str_set(&shm.name, "nginx_shared_zone");
shm.log = cycle->log;

if (ngx_shm_alloc(&shm) != NGX_OK) {
    return NGX_ERROR;
}

shared = shm.addr;
...
ngx_stat_accepted = (ngx_atomic_t *) (shared + 3 * cl);
ngx_stat_handled = (ngx_atomic_t *) (shared + 4 * cl);
ngx_stat_requests = (ngx_atomic_t *) (shared + 5 * cl);
ngx_stat_active = (ngx_atomic_t *) (shared + 6 * cl);
ngx_stat_reading = (ngx_atomic_t *) (shared + 7 * cl);
ngx_stat_writing = (ngx_atomic_t *) (shared + 8 * cl);
ngx_stat_waiting = (ngx_atomic_t *) (shared + 9 * cl);

```


## 2. 使用slab 算法操作共享内存

### 2.1 slab 算法的经典步骤

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

## 3. 比较常见的场景

### 3.0 选哪种共享进程的方式
atomic 适合简单计数

slab 适合key value，扩展性更好

### 3.1. 限流场景
限流的key 是location，value 是计数

### 3.2. 策略中心
红黑树的key 是策略名,红黑树的data 是策略，可以放一个json,策略可以增删改查，但是操作需要加锁。动态变更可以暴露api接口。



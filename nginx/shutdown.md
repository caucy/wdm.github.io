# nginx 定时器导致进程不退出分析

## nginx 进程为什么一直shutdown不退出
ngx_worker_process_cycle 在每个循环的时候，都会检查ngx_exiting，判断是否有定时器没退出
```
  if (ngx_exiting){
    if(ngx_event_no_timers_left()==NGX_OK){
      ngx_worker_process_exit();
    }
  }
```

而ngx_event_no_timers_left 会判断是否剩下不可cancle 的定时器，如果有，会不让退出
```
ngx_int_t
ngx_event_no_timers_left(void)
{
    ngx_event_t        *ev;
    ngx_rbtree_node_t  *node, *root, *sentinel;

    sentinel = ngx_event_timer_rbtree.sentinel;
    root = ngx_event_timer_rbtree.root;

    if (root == sentinel) {
        return NGX_OK;
    }

    for (node = ngx_rbtree_min(root, sentinel);
         node;
         node = ngx_rbtree_next(&ngx_event_timer_rbtree, node))
    {
        ev = ngx_rbtree_data(node, ngx_event_t, timer);

        if (!ev->cancelable) {
            return NGX_AGAIN;
        }
    }

    /* only cancelable timers left */

    return NGX_OK;
}
```
所有，对于一些可以随时退出的定时器，可以直接设置cancelable 为1

## 如何定位被哪个定时器给hang住了

gdb 可以直接挂到ngx_event_no_timers_left，打印出root 的具体信息，看handler 是哪个回调函数，哪些定时器是cancelable 0 的。

## 如何解决定时器退出问题
方式一，设置定时器是cancleable, 在进程退出时将不再阻塞进程退出

方式二，控制边界条件，正常del 定时器

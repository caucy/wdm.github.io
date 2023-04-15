# nginx编译成静态库写单测

## nginx 编译成静态库的步骤

1. replace main 函数

main 名字要换了，否则生成可执行文件会出错

2. 编译所有.c 文件

./configure & make 将生成所有.o 文件到obj/src 下，最后生成nginx 二进制会报错，可以忽略，每个c 文件都会生产对应的.o 文件。


3. 将.o 文件合并成.a 文件


ar -rc libnginx.a `find objs -name *.o` # 编译的文件可以合并成静态库


4. 准备头文件，使用静态库


```
cp `find src -name *.h` /usr/include/nginx/ # 注意windows 文件反复覆盖问题

cp objs/*.h /usr/include/nginx/ # 拷贝modules 下 .h文件

```

## 使用nginx 静态库写功能函数

可以写demo 跑
```
#include <ngx_core.h>

int main(){
    ngx_rbtree_t      tree;      //声明红黑树
    ngx_rbtree_node_t sentinel;  //声明哨兵节点
 
    //初始化
    ngx_rbtree_init(&tree, &sentinel, ngx_str_rbtree_insert_value);
 
    ngx_str_node_t strNode[5];
 
    ngx_str_set(&strNode[0].str, "abc0");
    strNode[0].node.key = ngx_crc32_long(strNode[0].str.data, strNode[0].str.len);  
 
    ngx_str_set(&strNode[1].str, "abc1");
    strNode[1].node.key = ngx_crc32_long(strNode[1].str.data, strNode[1].str.len);
 
    ngx_str_set(&strNode[2].str, "abc2");
    strNode[2].node.key = ngx_crc32_long(strNode[2].str.data, strNode[2].str.len);

    ngx_str_set(&strNode[3].str, "abc35");
    strNode[3].node.key = ngx_crc32_long(strNode[3].str.data, strNode[3].str.len);
 
    ngx_str_set(&strNode[4].str, "abd4");
    strNode[4].node.key = ngx_crc32_long(strNode[4].str.data, strNode[4].str.len);

    //遍历加入到红黑树
    ngx_int_t i;
    for (i = 0; i < 5; ++i){
		  ngx_rbtree_insert(&tree, &strNode[i].node);
    }

	ngx_str_node_t *tmpnode;
    //查找红黑树中最小节点
    tmpnode = (ngx_str_node_t *)ngx_rbtree_min(tree.root, &sentinel);

    //超找特定的字符串
    //tmpnode = (ngx_str_node_t *)ngx_str_rbtree_lookup(&tree, &strNode[0].str, 8);
}
```

编译:
```
gcc main.c -lnginx -lpthread -lpcre -lssl -lcrypto -lz -lcrypt -ldl -I /usr/include/nginx/
```


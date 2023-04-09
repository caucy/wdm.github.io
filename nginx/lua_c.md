# nginx 中lua 调用c的方法

## 1，通过ngx.var 的方式取nginx variable

实现varible 的所有方法，lua 中通过ngx.var.xxx 调用
例子：在c 模块中实现geoip module 中variable的读写，可以在lua 中通过ngx.var.xxx 读写变量。
```
static ngx_stream_variable_t  ngx_stream_geoip_vars[] = {

    { ngx_string("geoip_country_code"), NULL,
      ngx_stream_geoip_country_variable,
      NGX_GEOIP_COUNTRY_CODE, 0, 0 },
}
```

## 2，通过c so 调用c代码
比较常见的在openresty 中调用cjson库等

### 1、说明
1、Lua利用一个虚拟的堆栈来给C传递值或从C获取值。
2、当Lua调用C函数，都会获得一个新的堆栈，该堆栈初始包含所有的调用C函数所需要的参数值（Lua传给C函数的调用实参）。
3、C函数执行完毕后，会把返回值压入这个栈（Lua从中拿到C函数调用结果）。

### 2、举例
c测试代码
```
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>

static int c_test(lua_State* L)
{
    const char *a = luaL_checkstring(L, 1); //取栈第一个参数
    lua_pushstring(L, (const char *)a);//返回值入栈
    return 1;
}

static const struct luaL_Reg reg_test[] = {
    {"lua_test", c_test},
    {NULL, NULL}
};

int luaopen_hello(lua_State *L) {
    // 新建一个库(table)，把函数加入这个库中，并返回
    // lua_newtable(L);
    //luaL_setfuncs(L, reg_test, 0);
    const char* libName = "testlib";
    luaL_register(L, libName, reg_test);
    return 1;
}
```

这里可以用luaL_register 注册，也可以用Lua_newlib 注册，也可以用lua_pushcfunction
这个函数名有个命名规则，前缀为luaopen，后面就是lua中require的字符串。当执行到require "lib.test"时（假设将so库放于工作目录下的lib目录中），lua解析器会去lib/test.so文件中寻找并执行函数名为luaopen_lib_test的函数，找不到则报错。

## 3,  挂载函数到package.preload
### 3.1. preload 的原理
ngx_http_lua_upstream_module 有完整的demo
下面是参数例子：
```
static int
ngx_http_lua_upstream_create_module(lua_State * L)
{
    lua_createtable(L, 0, 6);

    lua_pushcfunction(L, ngx_http_lua_upstream_get_upstreams);
    lua_setfield(L, -2, "get_upstreams");

    lua_pushcfunction(L, ngx_http_lua_upstream_get_servers);
    lua_setfield(L, -2, "get_servers");

    lua_pushcfunction(L, ngx_http_lua_upstream_get_primary_peers);
    lua_setfield(L, -2, "get_primary_peers");

    lua_pushcfunction(L, ngx_http_lua_upstream_get_backup_peers);
    lua_setfield(L, -2, "get_backup_peers");

    lua_pushcfunction(L, ngx_http_lua_upstream_set_peer_down);
    lua_setfield(L, -2, "set_peer_down");

    lua_pushcfunction(L, ngx_http_lua_upstream_current_upstream_name);
    lua_setfield(L, -2, "current_upstream_name");

    return 1;
}
```

require 的顺序最先找的就是package.preload  table，所以可以将自己的模块封装成一个table，key 为函数名，value 为c函数签名
```
The first searcher simply looks for a loader in the package.preload table.

The second searcher looks for a loader as a Lua library, using the path stored at package.path. A path is a sequence of templates separated by semicolons. For each template, the searcher will change each interrogation mark in the template by filename, which is the module name with each dot replaced by a "directory separator" (such as "/" in Unix); then it will try to open the resulting file name. So, for instance, if the Lua path is the string

     "./?.lua;./?.lc;/usr/local/?/init.lua"
the search for a Lua file for module foo will try to open the files ./foo.lua, ./foo.lc, and /usr/local/foo/init.lua, in that order.

The third searcher looks for a loader as a C library, using the path given by the variable package.cpath. For instance, if the C path is the string

     "./?.so;./?.dll;/usr/local/?/init.so"
```

### 3.2. nginx 的封装ngx_http_lua_add_package_preload
ngx_http_lua_add_package_preload 将request 放到Luastat 中了，可以方便的取出request

## 4, 如何选择交互方式？
如果命令跟request 相关，可以使用ngx_http_lua_add_package_preload 加挂载点，如果有现成的变量可以使用ngx.var， 如果完全没关系，类似cjson 可以使用luaL_register 注册函数

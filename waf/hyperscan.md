# hyperscan 的使用

## hyperscan 简介
waf 需要对流量进行多次正则匹配，判断是否存在sql注入，命令注入等等。正则匹配非常耗费cpu，hyperscan 就是一个非常高性能的正则表达式引擎，可以对字符串，对stream做
多次正则匹配。

## hyperscan 作为正则表达式引擎的使用
hyperscan常见的api 比较简单：
### 编译规则
hs_compile() 将单个规则编译成数据库。
hs_compile_multi() 将一组规则编译成数据库。
hs_compile_ext_multi() 将一组规则编译成数据库，并支持扩展参数。具体细节可以参考《深入浅出 Hyperscan: 高性能正则表达式算法原理与设计》

### 扫描数据
hs_scan()进行匹配，调用时需要规定回调函数

/** 参数说明
* 1. 数据库
* 2. 待匹配字符串
* 3. 字符串长度
* 4. flag， 未定义，官方说明待用
* 5. scratch
* 6. 回调函数
* 7. 回调函数传参
*/
hs_scan(HsDatabase, tmpString, strlen(tmpString), 0, HsStra, onMatch, tmpString);

### 回调函数
/**
 * 回调函数
 * @param id
 * 规则 id
 * @param from
 * 指向匹配起始位置
 * @param to
 * 匹配结束位置
 * @param flags
 * 根本没用, hyperscan 官方明确说明没有使用的参数。
 * @param ctx
 * 输入的原始数据
 * @return
 * 暂时没用
 */
static int onMatch(unsigned int id, unsigned long long from, unsigned long long to, unsigned int flags, void *ctx) {}


## hyperscan 的用法
伪代码如下：
```
 // 准备规则id，规则表达式，规则flag 到一个数组
 int i;
 for (i = 0; i < count; i++) {
     compilePattern.ids[i] = CompPatt[i].ids;
     compilePattern.exps[i] = CompPatt[i].exps;
     compilePattern.flag[i] = HS_FLAG_CASELESS;   // 忽略大小写
 }

// 编译规则成database
 error = hs_compile_multi(compilePattern.exps, compilePattern.flag, compilePattern.ids, count, HS_MODE_BLOCK, NULL,
                          pDatabase,
                          (hs_compile_error_t **) &ComError);
 if (error != HS_SUCCESS) {
     // 失败处理
 }

// 扫描数据
 hs_scan(HsDatabase, tmpString, strlen(tmpString), 0, HsStra, onMatch, tmpString);

```

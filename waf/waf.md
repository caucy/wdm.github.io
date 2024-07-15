# 一些常见的攻击和waf拦截手段

waf 核心逻辑一般就是对query，body 做多正则匹配。

## sql注入
对query，body 做正则匹配
```
select.+(from|limit) // 含有select
(?:(union(.*?)select)) // 含有union
(?:from\W+information_schema\W) // 查mysql information_schema表
into(\s+)+(?:dump|out)file\s* // 读写文件
(order|group)\s+by.+ //查表字段
(?:alter\s*\w+.*character\s+set\s+\w+) //改字符集
(?:(?:select|create|rename|truncate|alter|delete|update|insert|desc)\s+(?:(?:group_)concat|char|load_file)\s?\(?)|(?:end\s*\);)
。。。。
```


## 拦截方式
https://github.com/heartshare/waf-6/blob/master/rules/cmdInjectList.rule

## 命令注入

## 拦截方式

## csrf攻击

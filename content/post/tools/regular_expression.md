---
title: regular expression
date: '2025-12-28T09:38:23+08:00'

categories:
- tool
---
# 正则表达式

正则表达式官网：(https://regex101.com/)

**基础常用语法**

```javascript
abc  //匹配字符abc

^a	//以a开头,匹配单个字符a，^开头

[a-z]	//匹配字符a-z的任意一个或多个字符

[^a-z]	//取反 除a-z字符

^[a-z]$	//匹配a-z任意一个字符开头且结尾 $结尾

.	//匹配除换行符（\n、\r）之外的任何单个字符，相等于 [^\n\r]。

\d	//匹配数字

\D	//匹配除数字

\w	//匹配数字字符下划线 相当于[a-zA-Z0-9_]，\W取反

^\w*	//*零个到多个

^\w+	//+一个到多个
    
^[a-z]{1，}$	//{1，}一个到多个 相当于 + 但不包含回车
 
\s	//相当于 [\r\n\t\f\v ]

^[a-zA-Z0-9]\w*@qq.com  //邮箱匹配  

a|b	//匹配a，b字符
    
```



**进阶group入门语法**

![image-20240104132952327](/img/tool/regular_expression/image-20240104132952327.png)





![image-20240104133906900](/img/tool/regular_expression/image-20240104133906900.png)



foo(?=bar)   ==foo==bar,fooboo  //匹配bar前面的foo

foo(?!bar)   foobar,==foo==boo  //匹配bar前面的foo

```javascript
\k<a> matches the same text as most recently matched
^(?<a>.)\k<a>(?!\k<a>)(?<b>.)(?!\k<b>|\k<a>).$  //匹配aabc
```


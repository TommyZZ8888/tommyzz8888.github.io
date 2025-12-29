---
title: lua
date: '2025-12-28T09:38:23+08:00'

categories:
- tool
---
```lua
-- 直接赋值是全局变量一般不要这么使用
a = 10 
-- local是指定当前作用域（一般指大括号或当前文件内）有效的变量
local a = 10

-- 数字
local b = 10.1
print(a + b) -- 打印20.1，支持常见的运算符以及位运算符
print(15 & (1<<2)) -- 4

-- 布尔
local a = false
local b = true
print(a and b) -- 打印false
print(a or b)  -- 打印true
print(not a)   -- 打印true

-- 字符串
local a = "abc"
local b = 'def' -- 单双引号皆可
print(a..b) -- 字符串拼接 abcdef
print(#a)   -- 字符串长度 3
print(string.upper(a)) -- 使用string库封装的函数，转大写 ABC

-- nil
local a = nil
if not a then print("a is", false) end

-- 函数
function f1(a)
    print(a)
    return 1,"abc" -- 支持多返回值
end
local r1, r2 = f1('param') -- r1 r2分别对应两个返回值，只写一个就只拿第一返回值

local f2 = function() return 1 end -- 函数也是一种数据类型（闭包），可以直接赋值给变量
local r1 = f2()

-- table模拟数组
local arr = {1, "abc", 2, function() print('hello') end} -- 数组
print(arr[1]) -- 打印1
table.insert(arr,'haha') -- 在arr最后插入元素
table.insert(arr, 1, 'haha') -- 在arr指定下标插入元素，后面元素依次后移
table.remove(arr, 1) -- 删除指定下标元素
-- 遍历数组元素
for k, v in pairs({'a', 'b'}) do
    print(k, v) -- 打印1 a  2 b
end

-- table模拟字典或对象
local dict = {a=1, b=2}
print(dict.a) -- 等价于print(dict['a'])
dict.c = 3   -- 新增元素
dict.a = nil -- 删除元素
-- 遍历字典元素
for k, v in pairs({a=1, b=2}) do
    print(k, v) -- 打印a 1  b 2
end

-- 字符串函数 string.xx(字符串，其他参数) 语法糖：可以简写为 字符串变量名:xx(其他参数)
local a = "abc"
local a = string.char(0xfe, 0xff)
print(a) -- 俩乱码，因为ascii只有前128个字符，后面的都是乱码
print(a:byte(1)) -- ！！！但是与其他语言不通，lua中的乱码是无损的，是可以反向转为原始的byte 254
print(a:byte(2)) -- ！！！这个特点也使得，lua常用string无损的存储网络流byte数组，弥补了byte[]类型的缺失 255
print(string.upper("abc"))
print(a:upper())      -- 转大写，与上一行等价
print(a:lower())
local a = "abc123abc123"
print(a:find("23")) -- 找到第一个23的位置，返回值是两个，开始和结束的下标5 6【lua中下标从1开始】
print(a:find("%d%d", 7))  -- find参数是正则，与传统正则稍有区别的是\改为%。第二参数是开始寻找的下标 打印10 11
print(a:match("%d%d", 7)) -- match与find唯一区别是，find打印下标，match打印匹配的部分的字符串 打印12
print(a:reverse())  -- 翻转
print(a:sub(2,-2)) -- 截取，左闭右闭区间 打印bc123abc12
print(a:gsub("123","def")) -- 替换所有的123为def，有两个返回值是替换后的字符串和替换了多少个，打印abcdefabcdef 2
print(string.format('num is %d', 4)) -- 类似c中printf

-- 判断
local a = 10>1
local b = 20>13
if a and b then
    print("a is true and b is true")
elseif a then
    print("only a is true")
else
    print("only b is true")
end

-- 循环
local a = 0
while(a < 10) do
   print(a) --打印0-9
   a = a + 1 -- lua不支持++ 和+=
end

for i=1,10,2 do -- 1到10，步长2
    print(i) --打印13579
end

-- 0是true
if 0 then print('0 = true') end
if not nil then print('nil = false') end
-- 下标从1开始, sub左闭右闭
print(string.sub("abc", 1,2)) -- ab
```


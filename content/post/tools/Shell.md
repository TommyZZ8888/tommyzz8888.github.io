---
title: shell
date: '2025-12-28T09:38:23+08:00'

categories:
- tool
---
## Shell

### 入门

#### 概念

Shell 脚本（shell script），是一种为 shell 编写的脚本程序，又称 Shell 命令稿、程序化脚本，是一种计算机程序使用的文本文件，内容由一连串的 shell 命令组成，经由 Unix Shell 直译其内容后运作

Shell 被当成是一种脚本语言来设计，其运作方式与解释型语言相当，由 Unix shell 扮演命令行解释器的角色，在读取 shell 脚本之后，依序运行其中的 shell 命令，之后输出结果



#### 环境

Shell 编程跟 JavaScript、php 编程一样，只要有一个能编写代码的文本编辑器和一个能解释执行的脚本解释器就可以了。

`cat /etc/shells`：查看解释器
![](..\img\tool\shell\shell环境.png)

Linux 的 Shell 种类众多，常见的有：

- Bourne Shell（/usr/bin/sh或/bin/sh）
- Bourne Again Shell（/bin/bash）：Bash 是大多数Linux 系统默认的 Shell
- C Shell（/usr/bin/csh）
- K Shell（/usr/bin/ksh）
- Shell for Root（/sbin/sh）

- 等等……



#### 第一个shell

* 新建 s.sh 文件：touch s.sh

* 编辑 s.sh 文件：vim s.sh

  ```shell
  #!/bin/bash  --- 指定脚本解释器
  echo "你好，shell !"   ---向窗口输入文本
  
  :<<!
  写shell的习惯 第一行指定解释器
  文件是sh为后缀名
  括号成对书写
  注释的时候尽量不用中文注释。不友好。
  [] 括号两端要要有空格。  [ neirong ]
  习惯代码索引，增加阅读性
  写语句的时候，尽量写全了，比如if。。。
  !
  ```

* 查看 s.sh文件：ls -l    s.sh文件权限是【-rw-rw-r--】

* chmod a+x s.sh         s.sh文件权限是【-rwxrwxr-x】

* 执行文件：./s.sh

* 或者直接  `bash s.sh`

**注意：**

**#!** 是一个约定的标记，告诉系统这个脚本需要什么解释器来执行，即使用哪一种 Shell

echo 命令用于向窗口输出文本



***



### 注释

* 单行注释：以 **#** 开头的行就是注释，会被解释器忽略

* 多行注释：

  ```shell
  :<<EOF
  注释内容...
  注释内容...
  EOF
  ```

  ```shell
  :<<!      -----这里的符号要和结尾处的一样
  注释内容...
  注释内容...
  注释内容...
  !        
  ```



***



### 变量

#### 定义变量

变量名和等号之间不能有空格，这可能和你熟悉的所有编程语言都不一样。同时，变量名的命名须遵循如下规则：

- 命名只能使用英文字母，数字和下划线，首个字符不能以数字开头。
- 中间不能有空格，可以使用下划线（_）。
- 不能使用标点符号。
- 不能使用bash里的关键字（可用help命令查看保留关键字）。



  

####  使用变量

 使用一个定义过的变量，只要在变量名前面加美元符号$即可

```shell
name="seazean"
echo $name
echo ${name}
name="zhy"
```

* 已定义的变量，可以被重新定义变量名

* 外面的花括号是可选的，加不加都行，加花括号是为了帮助解释器识别变量的边界。推荐加！！

  ```sh
  比如：echo "I am good at ${shell-t}Script"
  通过上面的脚本我们发现，如果不给shell-t变量加花括号，写成echo "I am good at $shell-tScript"，解释器shell就会把$shell-tScript当成一个变量，由于我们前面没有定义shell-t变量，那么解释器执行执行的结果自然就为空了。
  ```

  



#### 只读变量

使用 readonly 命令可以将变量定义为只读变量，只读变量的值不能被改变。(类似于final)

```shell
#!/bin/bash
myUrl="https://www.baidu.com"
readonly myUrl
myUrl="https://cn.bing.com/"  
#报错 myUrl readonly
```



#### 删除变量

使用 unset 命令可以删除变量，变量被删除后不能再次使用。

语法：`unset variable_name`

```shell
#!/bin/sh
myUrl="https://www.baidu.com"
unset myUrl
echo $myUrl
```

定义myUrl变量，通过unset删除变量，然后通过echo进行输出，**结果是为空**，没有任何的结果输出。

  

#### 字符变量

> 字符串是shell编程中最常用也是最有用的数据类型，字符串可以用单引号，也可以用双引号，也可以不用引号，在Java SE中我们定义一个字符串通过Stirng  s=“abc" 双引号的形式进行定义，而在shell中也是可以的。

##### 引号

* **单引号**

  ```sh
  str='this is a string variable'
  ```

  单引号字符串的限制：
  * 单引号里的任何字符都会原样输出，单引号字符串中的**变量是无效的**；
  * 单引号字串中不能出现单独一个的单引号（对单引号使用转义符后也不行），但可成对出现，作为字符串拼接使用。

* 双引号

  ```shell
  your_name='frank'
  str="Hello,\"$your_name\"! \n"
  echo -e $str     #Hello, "frank"!
  ```

  双引号的优点：

  - 双引号里可以有变量
  - 双引号里可以出现转义字符



##### 拼接字符串

```shell
your_name="frank"
# 使用双引号拼接
greeting="hello, "$your_name" !"
greeting_1="hello, ${your_name} !"
echo $greeting  $greeting_1
#hello,frank! hello,frank
```



##### 获取字符串长度

命令：`${#variable_name}`

```sh
string="seazean"
echo ${#string} #7
```



##### 提取字符串

```shell
string="abcdefghijklmn"
echo ${string:1:4} 
```

输出为【bcde】，通过截取我们发现，它的下标和我们在java中的读取方式是一样的，下标也是从0开始。



***



### 数组

bash支持一维数组（不支持多维数组），并且没有限定数组的大小。

#### 定义数组

在 Shell 中，用括号来表示数组，数组元素用"空格"符号分割开

```shell
数组名=(值1 值2 ... 值n)
array_name=(value0 value1 value2 value3) 
array_name=(
value0
value1
value2
value3
)
```

通过下标定义数组中的其中一个元素：

```shell
array_name[0]=value0
array_name[1]=value1
array_name[n]=valuen
```

可以不使用连续的下标，而且下标的范围没有限制



#### 读取数组

读取数组元素值的一般格式是：

```sh
${数组名[下标]}

value=${array_name[n]}
echo ${value}
```

使用 **@** 符号可以获取数组中的所有元素，例如：`echo ${array_name[@]}`



#### 获取长度

获取数组长度的方法与获取字符串长度的方法相同，数组前加#

```sh
# 取得数组元素的个数
length=${#array_name[@]}
# 或者
length=${#array_name[*]}
```

```sh
#! /bin/bash
g=(a b c d e f)
echo "数组下标为2的数据为:" ${g[2]}  #c
echo  "数组所有数据为:"  ${#g[@]}   #6
echo  "数组所有数据为:"   ${#g[*]}  #6
```



***



### 运算符

Shell 和其他编程一样，**支持**包括：算术、关系、布尔、字符串等运算符。原生 bash **不支持 **简单的数学运算，但是可以通过其他命令来实现，例如expr。expr 是一款表达式计算工具，使用它能完成表达式的求值操作。

#### 规则

* **表达式和运算符之间要有空格**，例如 2+2 是不对的，必须写成 2 + 2
* **完整的表达式要被 `` 包含，注意不是单引号**
* **条件表达式要放在方括号之间，并且要有空格**，例如: `[$a==$b]` 是错误的，必须写成 `[ $a == $b ]`
* **(())双括号里可以跟表达式**，例如((i++))，((a+b))



#### 算术运算符

| **运算符** | **说明**                                      | **举例**                      |
| ---------- | --------------------------------------------- | ----------------------------- |
| +          | 加法                                          | `expr $a + $b` 结果为 30。    |
| -          | 减法                                          | `expr $a - $b` 结果为 -10。   |
| *          | 乘法                                          | `expr $a \* $b` 结果为  200。 |
| /          | 除法                                          | `expr $b / $a` 结果为 2。     |
| %          | 取余                                          | `expr $b % $a` 结果为 0。     |
| =          | 赋值                                          | a=$b 将把变量 b 的值赋给 a。  |
| ==         | 相等。用于比较两个数字，相同则返回 true。     | `[ $a == $b ] `返回 false。   |
| !=         | 不相等。用于比较两个数字，不相同则返回 true。 | `[ $a != $b ] `返回 true。    |

```shell
#! /bin/bash
a=4
b=20
echo "加法运算"  `expr $a + $b` 
echo "乘法运算，注意*号前面需要反斜杠" ` expr $a \* $b`
echo "加法运算"  `expr  $b / $a`
((a++))
echo "a = $a"
c=$((a + b)) 
d=$[a + b]
echo "c = $c"
echo "d = $d"
```

```
//结果
加法运算 24
减法运算 -16
乘法运算，注意*号前面需要反斜杠 80
加法运算 5
a = 5
c = 25
d = 25
```





#### 字符运算符

假定变量 a 为 "abc"，变量 b 为 "efg"，true=0，false=1。

| 运算符 | 说明                                      | 举例                       |
| :----- | :---------------------------------------- | :------------------------- |
| =      | 检测两个字符串是否相等，相等返回 true。   | `[ $a = $b ]` 返回 false。 |
| !=     | 检测两个字符串是否相等，不相等返回 true。 | `[ $a != $b ]` 返回 true。 |
| -z     | 检测字符串长度是否为0，为0返回 true。     | `[ -z $a ]` 返回 false。   |
| -n     | 检测字符串长度是否为0，不为0返回 true。   | `[ -n "$a" ]` 返回 true。  |
| $      | 检测字符串是否为空，不为空返回 true。     | `[ $a ]` 返回 true。       |

```shell
a="abc"
b="efg"

if [ $a = $b ]
then
   echo "$a = $b : a 等于 b"
else
   echo "$a = $b: a 不等于 b"
fi
if [ $a != $b ]
then
   echo "$a != $b : a 不等于 b"
else
   echo "$a != $b: a 等于 b"
fi
```





#### 关系运算符

关系运算符只支持数字，不支持字符串，除非字符串的值是数字。

下表列出了常用的关系运算符，假定变量 a 为 10，变量 b 为 20：

| 运算符 | 说明                                                  | 举例                         |
| :----- | :---------------------------------------------------- | :--------------------------- |
| -eq    | 检测两个数是否相等，相等返回 true。                   | `[ $a -eq $b ]` 返回 false。 |
| -ne    | 检测两个数是否不相等，不相等返回 true。               | `[ $a -ne $b ]` 返回 true。  |
| -gt    | 检测左边的数是否大于右边的，如果是，则返回 true。     | `[ $a -gt $b ]` 返回 false。 |
| -lt    | 检测左边的数是否小于右边的，如果是，则返回 true。     | `[ $a -lt $b ]` 返回 true。  |
| -ge    | 检测左边的数是否大于等于右边的，如果是，则返回 true。 | `[ $a -ge $b ]` 返回 false。 |
| -le    | 检测左边的数是否小于等于右边的，如果是，则返回 true。 | `[ $a -le $b ]` 返回 true。  |

```sh
a=10
b=20

if [ $a -eq $b ]
then
   echo "$a -eq $b : a 等于 b"
else
   echo "$a -eq $b: a 不等于 b"
fi
```





#### 布尔运算符

下表列出了常用的布尔运算符，假定变量 a 为 10，变量 b 为 20：

| 运算符 | 说明                                                | 举例                               |
| :----- | :-------------------------------------------------- | :--------------------------------- |
| !      | 非运算，表达式为 true 则返回 false，否则返回 true。 | [ ! false ] 返回 true。            |
| -o     | 或运算，有一个表达式为 true 则返回 true。           | `[ $a -lt 20 -o $b -gt 100 ]`true  |
| -a     | 与运算，两个表达式都为 true 才返回 true。           | `[ $a -lt 20 -a $b -gt 100 ]`false |



#### 逻辑运算符

假定变量 a 为 10，变量 b 为 20:

| 运算符 | 说明       | 举例                                         |
| :----- | :--------- | :------------------------------------------- |
| &&     | 逻辑的 AND | `[[ $a -lt 100 && $b -gt 100 ]]` 返回 false  |
| \|\|   | 逻辑的 OR  | `[[ $a -lt 100 \|\| $b -gt 100 ]]` 返回 true |



***



### 流程控制

#### if

```shell
if condition
then
    command1 
    command2
    ...
    commandN 
fi
#末尾的fi就是if倒过来拼写
```

```shell
if condition
then
    command1 
    command2
    ...
    commandN
else
    command
fi
```

```shell
if condition1
then
    command1
elif condition2 
then 
    command2
else
    commandN
fi
```

* 查找一个进程，如果进程存在就打印true

  ```shell
  if [ $(ps -ef | grep -c "ssh") -gt 1 ]]
  then 
  	echo "true"
  fi
  ```

* 判断两个变量是否相等

  ```shell
  a=10
  b=20
  if [ $a == $b ]
  then
     echo "a 等于 b"
  elif [ $a -gt $b ]
  then
     echo "a 大于 b"
  elif [ $a -lt $b ]
  then
     echo "a 小于 b"
  else
     echo "没有符合的条件"
  fi
  ```

  



#### for

for循环格式为：

```shell
for var in item1 item2 ... itemN
do
    command1
    command2
    ...
    commandN
done
```

顺序输出当前列表中的字母：

```shell
for loop in A B C D E F G 
do
    echo "顺序输出字母为: $loop"
done

顺序输出字母为:A
顺序输出字母为:B
....
顺序输出字母为:G
```



#### while

while循环用于不断执行一系列命令，也用于从输入文件中读取数据 

```shell
while condition
do
    command
done
```

需求：如果int小于等于10，那么条件返回真。int从0开始，每次循环处理时，int加1。 

```shell
#!/bin/bash
a=1
while [ "${a}" -le 10 ]
do
    echo "输出的值为：" $a
    ((a++))
done
输出的值为：1
输出的值为：2
...
输出的值为：10
```



#### case...esac

与 switch ... case 语句类似，是一种多分枝选择结构，每个 case 分支用右圆括号开始，用两个分号 **;;** 表示 break，即执行结束，跳出整个 case ... esac 语句，esac（就是 case 反过来）作为结束标记。

```shell
case 值 in  
模式1)
    command1
    command2
    command3
    ;;
模式2）
    command1
    command2
    command3
    ;;
*)
    command1
    command2
    command3
    ;;
esac  #case反过来
```

* case 后为取值，值可以为变量或常数。

* 值后为关键字 in，接下来是匹配的各种模式，每一模式最后必须以右括号结束，模式支持正则表达式。

```shell
v="czbk"

case "$v" in
"czbk") 
	echo "传智播客"
   	;;
"baidu") 
	echo "baidu 搜索"
	;;
"google") 
	echo "google 搜索"
   	;;
esac
```





***



### 函数

#### 输入

函数语法如下：

```shell
[ function ] funname [()]
{
    action;
    [return int;]

}
```

- 1、可以使用function fun() 定义函数，也可以直接fun() 定义,不带任何参数。
- 2、函数参数返回，可以显示加：return 返回，如果不加，将以最后一条命令运行结果，作为返回值。 return后跟数值n(0-255

```shell
#无参无返回值的方法
method(){
	echo "函数执行了!"
}

#方法的调用
#method



#有参无返回值的方法
method2(){
	echo "接收到的第一个参数$1"
	echo "接收到的第二个参数$2"
}

#方法的调用
#method2 1 2

#有参有返回值方法的定义
method3(){
	echo "接收到的第一个参数$1"
	echo "接收到的第二个参数$2"
	return $(($1 + $2))
}

#方法的调用
method3 10 20
echo $?
```



#### 读取

`read 变量名` --- 表示把键盘录入的数据复制给这个变量

需求：在方法中键盘录入两个整数,返回这两个整数的和

```shell
method(){
	echo "请录入第一个数"
	read number1
	echo "请录入第二个数"
	read number2
	echo "两个数字分别为${number1},${number2}"
	return $((number1+number2))
}

method
echo $?
```



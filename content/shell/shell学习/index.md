+++
title = "shell学习"
date = "2026-05-28T00:01:08+08:00"
draft = false
+++

# 一、变量

普通变量与临时环境变量

普通变量定义：VAR=value

临时环境变量定义：export VAR=value

变量引用：$VAR

Shell 进程的环境变量作用域是 Shell 进程，当 export 导入到系统变量时，则作用域是 Shell 进程及其 Shell 子进程



## **常用的系统变量**

$SHELL ：默认 Shell

$HOME ：当前用户家目录

$IFS ：内部字段分隔符

$LANG ：默认语言

$PATH ：默认可执行程序路径

$PWD ：当前目录

$UID ：当前用户 ID

$USER ：当前用户

$HISTSIZE ：历史命令大小，可通过 HISTTIMEFORMAT 变量设置命令执行时间

$RANDOM ：随机生成一个 0 至 32767 的整数

$HOSTNAME ：主机名



## **位置变量**

位置变量指的是函数或脚本后跟的第 n 个参数。

$1-$n，需要注意的是从第 10 个开始要用花括号调用，例如${10}



## **特殊变量**

| 变量 | 解释                                             |
| ---- | ------------------------------------------------ |
| $0   | 脚本自身文件名                                   |
| $?   | 返回上一条命令执行是否成功，成功返回0，非0则失败 |
| $#   | 位置参数的总数                                   |
| #@   | 每个位置参数被看做独立的字符                     |
| $$   | 当前的进程                                       |
| $!   | 上一条运行后台进程的PID                          |
| $*   | 所有位置参数被看做一个字符串                     |



## 变量引用

```bash
VAR=123
echo $VAR
echo ${VAR} # 将{}视为一整个字符串，在有特殊字符情况下使用
```



**将命令的记过作为变量值：**

```bash
VAR=`echo 123`
echo $VAR
123

VAR=$(echo 123)
echo $VAR
123

#这里的反撇号等效于$()，都是用于执行 Shell 命令。
```



## 单引号与双引号

单引号是告诉 Shell 忽略特殊字符，而双引号则解释特殊符号原有的意义，比如$、！。

在变量赋值时，如果值有空格，Shell 会把空格后面的字符串解释为命令



## #{}的使用

### 1、输出字符串的值

```bash
VAR=123
echo ${VAR}
123
```



### 2、获取字符串的长度

```bash
VAR='hello world!'
echo $VAR
hello world!

echo ${#VAR}
12
```



### 3、字符串切片

格式：

​		${parameter:offset}：截取offset往后的字符

​		${parameter:offset:length}：截取从 offset 个字符开始，向后 length 个字符。

```bash
# 截取 hello 字符串：
VAR='hello world!'
echo ${VAR:0:5}
hello

# 截取 wo 字符：
echo ${VAR:6:2}
wo

# 截取 world!字符串：
echo ${VAR:5}
world!

# 截取最后一个字符：
echo ${VAR:(-1)}
!

# 截取最后二个字符：
echo ${VAR:(-2)}
d!

# 截取从倒数第 3 个字符后的 2 个字符：
echo ${VAR:(-3):2}
ld
```



### 4、替换字符

格式：

​		${parameter/pattern/string}

​		${parameter//pattern/string} ：# patterm 前面开头一个正斜杠为只匹配第一个字符串，两个正斜杠为匹配所有字符。

``` bash
VAR='hello world world!'
# 将第一个 world 字符串替换为 WORLD：
echo ${VAR/world/WORLD}
hello WORLD world!

# 将全部 world 字符串替换为 WORLD：
# patterm 前面开头一个正斜杠为只匹配第一个字符串，两个正斜杠为匹配所有字符。
echo ${VAR//world/WORLD}
hello WORLD WORLD!

# 替换正则匹配为空：
VAR=123abc
echo ${VAR//[^0-9]/}
123
echo ${VAR//[0-9]/}
abc
```



### 5、字符截取

格式：

${parameter#word} # 删除匹配前缀

${parameter##word} 

${parameter%word} # 删除匹配后缀

${parameter%%word}

\# 去掉左边，最短匹配模式，##最长匹配模式。

% 去掉右边，最短匹配模式，%%最长匹配模式。

```bash
URL="http://www.baidu.com/baike/user.html"
# 以//为分隔符截取右边字符串：
echo ${URL#*//}
www.baidu.com/baike/user.html

# 以/为分隔符截取右边字符串：
echo ${URL##*/}
user.html

# 以//为分隔符截取左边字符串：
echo ${URL%%//*}
http:

# 以/为分隔符截取左边字符串：
echo ${URL%/*}
http://www.baidu.com/baike

# 以.为分隔符截取左边：
echo ${URL%.*}
http://www.baidu.com/baike/user

# 以.为分隔符截取右边：
echo ${URL##*.}
html
```



### 5、**变量状态赋值** 

${VAR:-string} 如果 VAR 变量为空则返回 string

${VAR:+string} 如果 VAR 变量不为空则返回 string

${VAR:=string} 如果 VAR 变量为空则重新赋值 VAR 变量值为 string

${VAR:?string} 如果 VAR 变量为空则将 string 输出到 stderr

```bash
# 如果变量为空就返回 hello world!：
VAR=
echo ${VAR:-'hello world!'}
hello world!

# 如果变量不为空就返回 hello world!：
VAR="hello"
echo ${VAR:+'hello world!'}
hello world!

# 如果变量为空就重新赋值：
VAR=
echo ${VAR:=hello}
hello
echo $VAR
hello

# 如果变量为空就将信息输出 stderr：
VAR=
echo ${VAR:?value is null}
-bash: VAR: value is null
```

### 6、字符串颜色

![image-20231117210858484](C:\Users\ANYYNA\AppData\Roaming\Typora\typora-user-images\image-20231117210858484.png)

格式：

```bash
\033[1;31;40m  # 1 是显示方式，可选。31 是字体颜色。40m 是字体背景颜色。
\033[0m  # 恢复终端默认颜色，即取消颜色设置。

```

```bash
#!/bin/bash
# 字体颜色
for i in {31..37}; do
echo -e "\033[$i;40mHello world!\033[0m"
done

# 背景颜色
for i in {41..47}; do
echo -e "\033[47;${i}mHello world!\033[0m"
done

# 显示方式
for i in {1..8}; do
echo -e "\033[$i;31;40mHello world!\033[0m"
done
```



# 二、表达式与运算符

## 整数

| 比较符                | **描述**   | 示例                |
| --------------------- | ---------- | ------------------- |
| -eq，equal            | 等于       | [ 1 -eq 1 ] 为 true |
| -ne，not equal        | 不等于     | [ 1 -ne 1 ]为 false |
| -gt，greater than     | 大于       | [ 2 -gt 1 ]为 true  |
| -lt，lesser than      | 小于       | [ 2 -lt 1 ]为 false |
| -ge，greater or equal | 大于或等于 | [ 2 -ge 1 ]为 true  |
| -le，lesser or equal  | 小于或等于 | [ 2 -le 1 ]为 false |



<img src="C:\Users\ANYYNA\AppData\Roaming\Typora\typora-user-images\image-20231117212523460.png" alt="image-20231117212523460" style="zoom:67%;" />

$(())表达式还有一个用途，三目运算：

```bash
# 如果条件为真返回 1，否则返回 0
echo $((1<0))
0

echo $((1>0))
1

# 指定输出数字：
echo $((1>0?1:2))
1
echo $((1<0?1:2))
2
# 注意：返回值不支持字符串
```



## 字符串比较

<img src="C:\Users\ANYYNA\AppData\Roaming\Typora\typora-user-images\image-20231117212119536.png" alt="image-20231117212119536" style="zoom:50%;" />



示例：

```bash
# [ -z $a ] && echo yes || echo no
yes

# [ -n $a ] && echo yes || echo no
yes

# 加了双引号才能正常判断是否为空
# [ -z "$a" ] && echo yes || echo no
yes

# [ -n "$a" ] && echo yes || echo no
no

# 使用了双中括号就不用了双引号
# [[ -n $a ]] && echo yes || echo no
no
# [[ -z $a ]] && echo yes || echo no
yes
```



## 文件判断

<img src="C:\Users\ANYYNA\AppData\Roaming\Typora\typora-user-images\image-20231117212226747.png" alt="image-20231117212226747" style="zoom: 67%;" />



## 布尔运算符

<img src="C:\Users\ANYYNA\AppData\Roaming\Typora\typora-user-images\image-20231117212351687.png" alt="image-20231117212351687" style="zoom: 67%;" />



## 逻辑判断符

<img src="C:\Users\ANYYNA\AppData\Roaming\Typora\typora-user-images\image-20231117212424555.png" alt="image-20231117212424555" style="zoom:67%;" />



## **其他运算工具（let/expr/bc）** 

<img src="C:\Users\ANYYNA\AppData\Roaming\Typora\typora-user-images\image-20231117212745479.png" alt="image-20231117212745479" style="zoom:67%;" />



## shell括号用途总结

<img src="C:\Users\ANYYNA\AppData\Roaming\Typora\typora-user-images\image-20231117212836751.png" alt="image-20231117212836751" style="zoom:67%;" />



# 流程控制

## if语句

```bash
if 条件表达式; then
命令
fi



# 示例：
#!/bin/bash
N=10
if [ $N -gt 5 ]; then
echo yes
fi
# bash test.sh
yes
```

双分支

```bash
if 条件表达式; then
命令
else
命令
fi


# 示例
#!/bin/bash
N=10
if [ $N -lt 5 ]; then
	echo yes
else
	echo no
fi

```



多分支

```bash
if 条件表达式; then
	命令
elif 条件表达式; then
	命令
else
	命令
fi

# 示例
#!/bin/bash
N=$1
if [ $N -eq 3 ]; then
 echo "eq 3"
elif [ $N -eq 5 ]; then
 echo "eq 5"
elif [ $N -eq 8 ]; then
 echo "eq 8"
else
 echo "no"
fi
```

 

## for语句

```bash
for 变量名 in 取值列表; do
	命令
done

# 示例：
#!/bin/bash
for i in {1..3}; do
	echo $i
done

```

亦可以这么写

```bash
#!/bin/bash
for i in "$@"; {    # $@是将位置参数作为单个来处理
	echo $i
}
# bash test.sh 1 2 3
1
2
3
```



默认 for 循环的取值列表是以空白符分隔，也就是第一章讲系统变量里的$IFS

```bash
#!/bin/bash
for i in 12 34; do
	echo $i
done

# bash test.sh
12
34
```

如果想指定分隔符，可以重新赋值$FS变量

```bash
#!/bin/bash
OLD_IFS=$IFS
IFS=":"
for i in $(head -1 /etc/passwd); do
 echo $i
done
IFS=$OLD_IFS # 恢复默认值

# bash test.sh 
root
x
0
0
root
/root
/bin/bash
```



C语言风格：for (( expr1 ; expr2 ; expr3 )) ; do list ; done

示例

```bash
#!/bin/bash
for ((i=1;i<=5;i++)); do # 也可以 i--
	echo $i
done
```

```bash
#!/bin/bash
for ip in 192.168.1.{1..254}; do
 if ping -c 1 $ip >/dev/null; then
 	echo "$ip OK."
 else
 	echo "$ip NO!"
 fi
done
```



## while语句

```bash
while 条件表达式; do
	命令
done
```

示例

```bash
#!/bin/bash
N=0
while [ $N -lt 5 ]; do
	let N++
echo $N
done
# bash test.sh
1
2
3
4
5
```



**逐行读取文本**

方式1

```bash
#!/bin/bash
cat ./a.txt | while read LINE; do
	echo $LINE
done
```

方式2

```bash
#!/bin/bash
while read LINE; do
	echo $LINE
done < ./a.txt

```

方式3

```bash
#!/bin/bash
exec < ./a.txt # 读取文件作为标准输出
while read LINE; do
	echo $LINE
done
```



## continue 和 break

break 是终止循环。

continue 是跳出当前循环。

示例

```bash
#!/bin/bash
N=0
while true; do
	let N++
	if [ $N -eq 5 ]; then
		break
	fi
	echo $N
done
# bash test.sh
1
2
3
4
```

示例

```bash
#!/bin/bash
N=0
while [ $N -lt 5 ]; do
  let N++
  if [ $N -eq 3 ]; then
  	continue
  fi
  echo $N
done
```



## case

```bash
case 模式名 in
  模式 1)
  	命令
  ;;
  模式 2)
  	命令
  ;;
  *)
  	不符合以上模式执行的命令
esac
```

示例

```bash
#!/bin/bash
case $1 in
  start)
  	echo "start."
  ;;
  stop)
  	echo "stop."
  ;;
  restart)
  	echo "restart."
  ;;
  *)
  	echo "Usage: $0 {start|stop|restart}"
esac

# bash test.sh
Usage: test.sh {start|stop|restart}
# bash test.sh start
start.
# bash test.sh stop
stop.
# bash test.sh restart
restart.
```



# 三、函数和数组

## 函数

格式

```bash
func() {
	command
}

```

示例1

```bash
#!/bin/bash
func() {
	echo "This is a function."
}

func
# bash test.sh
This is a function.
```

示例2

```bash
#!/bin/bash
func() {
	VAR=$((1+1))
	return $VAR
	echo "This is a function."
}

func
echo $?
# bash test.sh

# return 在函数中定义状态返回值，返回并终止函数，但返回的只能是 0-255 的数字
```



## **传参**

```bash
#!/bin/bash
func() {
	echo "Hello $1"
}
func world

# bash test.sh
Hello world
```



## **数组**

数组是相同类型的元素按一定顺序排列的集合。

格式：

array=(元素 1 元素 2 元素 3 ...)

用小括号初始化数组，元素之间用空格分隔。

定义方法 1：初始化数组

array=(a b c)

定义方法 2：新建数组并添加元素

array[下标]=元素

定义方法 3：将命令输出作为数组元素

array=($(command))



数组操作

```bash
# 获取所有元素：
echo ${array[*]}  # *和@ 都是代表所有元素
a b c

# 获取元素下标：
echo ${!a[@]} 
0 1 2

# 获取数组长度：
echo ${#array[*]}
3

# 获取第一个元素：
echo ${array[0]}
a

# 获取第二个元素：
echo ${array[1]}
b

# 获取第三个元素：
echo ${array[2]}
c

# 添加元素：
array[3]=d
echo ${array[*]}
a b c d

# 添加多个元素：
array+=(e f g)
echo ${array[*]}
a b c d e f g

# 删除第一个元素：
unset array[0] # 删除会保留元素下标
echo ${array[*]}
b c d e f g

# 删除数组：
unset array
```



示例1

```bash
#!/bin/bash
for i in $(seq 1 10); do   # seq 生成数字序列
	array[a]=$i
	let a++
done
echo ${array[*]}

# bash test.sh
1 2 3 4 5 6 7 8 9 10
```

遍历数组元素

```bash
方法 1：
#!/bin/bash
IP=(192.168.1.1 192.168.1.2 192.168.1.3)
for ((i=0;i<${#IP[*]};i++)); do
	echo ${IP[$i]}
done

# bash test.sh
192.168.1.1
192.168.1.2
192.168.1.3

# 方法 2：
#!/bin/bash
IP=(192.168.1.1 192.168.1.2 192.168.1.3)
for IP in ${IP[*]}; do
	echo $IP
done
```



# 四、正则

<img src="C:\Users\ANYYNA\AppData\Roaming\Typora\typora-user-images\image-20231117215551515.png" alt="image-20231117215551515" style="zoom: 50%;" />

<img src="C:\Users\ANYYNA\AppData\Roaming\Typora\typora-user-images\image-20231117215621099.png" alt="image-20231117215621099" style="zoom: 50%;" />






































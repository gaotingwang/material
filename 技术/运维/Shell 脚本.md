## 一、shell 入门简介

### 1.1 什么是 shell

- **shell 脚本简介**

```
# 什么是shell
网上有很多shell 的概念介绍，其实都很官方化，如果你对linux 命令很熟悉，那么编写shell 就不是一个难事，shell 本质上是 linux 命令，一条一条命令组合在一起，实现某一个目的，就变成了shell脚本。它从一定程度上减轻了工作量，提高了工作效率。

# 官方化的shell 介绍
Shell 通过提示您输入，向操作系统解释该输入，然后处理来自操作系统的任何结果输出，简单来说Shell就是一个用户跟操作系统之间的一个命令解释器。

# 常见的shell 有哪些
Bourne Shell（/usr/bin/sh或/bin/sh）
Bourne Again Shell（/bin/bash）
C Shell（/usr/bin/csh）
K Shell（/usr/bin/ksh）
Shell for Root（/sbin/sh）
```

在一般情况下，人们并不区分 Bourne Shell 和 Bourne Again Shell，所以，像 `#!/bin/sh`，它同样也可以改为 `#!/bin/bash`。

`#!` 告诉系统其后路径所指定的程序即是解释此脚本文件的 Shell 程序。

### 1.2 shell 编程注意事项

- shell 命名：Shell 脚本名称命名一般为英文、大写、小写，后缀以. sh 结尾
- 不能使用特殊符号、空格
- 见闻之意，名称要写的一眼可以看出功能
- shell 编程 <font color="red">首行需要 #!/bin/bash 开头</font>
- shell 脚本 变量不能以 数字、特殊符号开头，可以使用下划线`_`, 但不能用破折号 `-`

### 1.3 第一个 shell 脚本 hello world

```sh
# 创建一个Helloword.sh 文件
[root@aly_server01~]# touch test.sh

# 编辑Helloword.sh 文件
[root@aly_server01~]# vim test.sh
[root@aly_server01~]# cat test.sh 
#!/bin/bash
# This is ower first shell
echo "hello world"
[root@aly_server01~]# 
[root@aly_server01~]# ll test.sh 
-rw-r--r-- 1 root root 85 Sep 20 22:26 test.sh

# 赋予执行权限
[root@aly_server01~]# chmod o+x test.sh 

# 运行helloword.sh 脚本
[root@aly_server01~]# ./test.sh 
hello world
```

一定要写成 `./test.sh`，而不是 `test.sh`，运行其它二进制的程序也一样，直接写 test.sh，linux 系统会去 PATH 里寻找有没有叫 test.sh 的，而只有 /bin, /sbin, /usr/bin，/usr/sbin 等在 PATH 里，你的当前目录通常不在 PATH 里，所以写成 test.sh 是会找不到命令的，要用 ./test.sh 告诉系统说，就在当前目录找。



## 二、shell 环境变量讲解

```sh
# 常见的3种变量
Shell编程中变量分为三种，分别是系统变量、环境变量和用户变量，Shell变量名在定义时，首个字符必须为字母（a-z，A-Z），不能以数字开头，中间不能有空格，可以使用下划线（_），不能使用（-），也不能使用标点符号等。

# 简单的变量介绍
[root@keeplived_server~]# a=18
[root@keeplived_server~]# echo $a
18
```

### 2.1 shell 系统变量介绍

```sh
# Shell常见的变量之一系统变量，主要是用于对参数判断和命令返回值判断时使用，系统变量详解如下：
$0   当前脚本的名称；
$n   当前脚本的第n个参数,n=1,2,…9；
$*   当前脚本的所有参数(不包括程序本身)； # 设在脚本运行时写了三个参数 1、2、3，则 "*" 等价于 "1 2 3"（传递了一个参数）
$@   当前脚本的所有参数(不包括程序本身)； # 设在脚本运行时写了三个参数 1、2、3，则 "@" 等价于 "1" "2" "3"（传递了三个参数）
$#   当前脚本的参数个数(不包括程序本身)；
$?   令或程序执行完后的状态，返回0表示执行成功，其他任何值表明有错误；
$$   程序本身的PID号
$!   后台运行的最后一个进程的ID号
$-   显示Shell使用的当前选项，与set命令功能相同
```

### 2.2 shell 环境变量介绍

```sh
#Shell常见的变量之二环境变量，主要是在程序运行时需要设置，环境变量详解如下：
PATH    命令所示路径，以冒号为分割；
HOME    打印用户家目录；
SHELL   显示当前Shell类型；
USER    打印当前用户名；
ID      打印当前用户id信息；
PWD     显示当前所在路径；
TERM    打印当前终端类型；
HOSTNAME    显示当前主机名；
PS1         定义主机命令提示符的；
HISTSIZE    历史命令大小，可通过 HISTTIMEFORMAT 变量设置命令执行时间;
RANDOM      随机生成一个 0 至 32767 的整数;
HOSTNAME    主机名
```

### 2.3 shell 用户自定义变量介绍

定义变量时，变量名不加美元符号；使用一个定义过的变量，只要在变量名前面加美元符号即可：

```shell
your_name="qinjx"
echo $your_name
echo ${your_name} # 变量名外面的花括号是可选的，加不加都行，加花括号是为了帮助解释器识别变量的边界
```

使用 `readonly` 命令可以将变量定义为只读变量，只读变量的值不能被改变：

```shell
#!/bin/bash

myUrl="https://www.google.com"
readonly myUrl
myUrl="https://www.runoob.com"  # 脚本运行结果：/bin/sh: NAME: This variable is read only.
```

使用 unset 命令可以删除变量，变量被删除后不能再次使用。unset 命令不能删除只读变量：

```shell
#!/bin/sh

myUrl="https://www.runoob.com"
unset myUrl
echo $myUrl # 实例执行将没有任何输出
```

字符串是shell编程中最常用最有用的数据类型（除了数字和字符串，也没啥其它类型好用了）。字符串可以用单引号，也可以用双引号，也可以不用引号：

```shell
your_name="runoob"
# 双引号里可以有变量,可以出现转义字符
str="Hello, I know you are \"$your_name\"! \n"
greeting="hello, "$your_name" !"       # 字符串拼接
greeting_1="hello, ${your_name} !"     # 双引号里可以有变量
echo $greeting  $greeting_1            # hello, runoob ! hello, runoob !

# 单引号里的任何字符都会原样输出，单引号字符串中的变量是无效的,对单引号使用转义符后也不行
greeting_2='hello, '$your_name' !'     # 字符串拼接
greeting_3='hello, ${your_name} !'     # 变量无效
echo $greeting_2  $greeting_3          # hello, runoob ! hello, ${your_name} !

# 获取字符串长度： 变量前加 # 用来获取长度
string="abcd"
echo ${#string}  # 4，等价于 ${#string[0]}

# 提取子字符串
string="runoob is a great site"
echo ${string:1:4} # 输出 unoo
```

bash支持一维数组（不支持多维数组），并且没有限定数组的大小，可以不使用连续的下标，而且下标的范围没有限制：

```shell
# 创建数组
my_array=(A B "C" D)
# 也可以使用数字下标来定义数组
array_name[0]=value0
array_name[1]=value1
array_name[2]=value2

# 读取数组： ${array_name[index]}
echo "第一个元素为: ${my_array[0]}"

# 关联数组
# 使用 declare 命令来声明，-A 选项就是用于声明一个关联数组，关联数组的键是唯一的
declare -A site=(["google"]="www.google.com" ["runoob"]="www.runoob.com" ["taobao"]="www.taobao.com")
# 也可以先声明一个关联数组，然后再设置键和值
declare -A sites
sites["google"]="www.google.com"

# 通过键来访问关联数组的元素
echo ${site["runoob"]}

# 获取数组中的所有元素：@ 或 * 可以获取数组中的所有元素
echo "数组的元素为: ${array_name[*]}" # 数组的元素为: value0 value1 value2
echo "数组的元素为: ${site[@]}"       # 数组的元素为: www.google.com www.runoob.com www.taobao.com

# 获取数组的所有键：在数组前加一个感叹号! 
echo "数组的键为: ${!site[*]}"        # 数组的键为: google runoob taobao

# 数组长度：与获取字符串长度的方法相同
echo "数组元素个数为: ${#my_array[*]}" # 数组元素个数为: 4
```

多行注释还可以使用以下格式：

```shell
:<<EOF
注释内容...
注释内容...
注释内容...
EOF

# EOF 也可以使用其他符号
:<<'
注释内容...
注释内容...
注释内容...
'

:<<!
注释内容...
注释内容...
注释内容...
!
```



## 三、shell 编程流程控制语句

### 3.1 运算符

Shell 和其他编程语言一样，支持多种运算符，包括：

- 算数运算符

  原生bash不支持简单的数学运算，但是可以通过其他命令来实现，例如 awk 和 expr，expr 最常用。expr 是一款表达式计算工具，使用它能完成表达式的求值操作

  ```shell
  #!/bin/bash
  
  # 注意使用的是反引号 `
  # 完整的表达式要被 ` ` 包含
  # 表达式和运算符之间要有空格
  val=`expr 2 + 2`
  echo "两数之和为 : $val"
  
  a=10
  b=20
  # 乘号(*)前边必须加反斜杠(\)才能实现乘法运算
  # 在 MAC 中 shell 的 expr 语法是：$((表达式))，此处表达式中的 "*" 不需要转义符号 "\"
  val=`expr $a \* $b`
  echo "a * b : $val"
  ```

  双小括号 `(( ))` 的语法格式为：`((表达式))`，通俗地讲，就是将数学运算表达式放在`((`和`))`之间。可以使用`$`获取 `(( ))` 命令的结果，这和使用`$`获得变量值是类似的。

  | 运算操作符/运算命令                            | 说明                                                         |
  | ---------------------------------------------- | ------------------------------------------------------------ |
  | ((a=10+66) <br />((b=a-15)) <br />((c=a+b))    | 这种写法可以在计算完成后给变量赋值。以 ((b=a-15)) 为例，即将 a-15 的运算结果赋值给变量 c。  注意，使用变量时不用加`$`前缀，(( )) 会自动解析变量名。 |
  | a=$((10+66) <br />b=$((a-15)) <br />c=$((a+b)) | 可以在 (( )) 前面加上`$`符号获取 (( )) 命令的执行结果，也即获取整个表达式的值。以 c=$((a+b)) 为例，即将 a+b 这个表达式的运算结果赋值给变量 c。  注意，类似 c=((a+b)) 这样的写法是错误的，不加`$`就不能取得表达式的结果。 |
  | ((a>7 && b==c))                                | (( )) 也可以进行逻辑运算，在 if 语句中常会使用逻辑运算。     |
  | echo $((a+10))                                 | 需要立即输出表达式的运算结果时，可以在 (( )) 前面加`$`符号。 |
  | ((a=3+5, b=a+10))                              | 对多个表达式同时进行计算。                                   |

- 关系运算符

  关系运算符只支持数字，不支持字符串，除非字符串的值是数字：

  | 运算符 | 说明                                                  |
  | :----- | :---------------------------------------------------- |
  | -eq    | 检测两个数是否相等，相等返回 true。                   |
  | -ne    | 检测两个数是否不相等，不相等返回 true。               |
  | -gt    | 检测左边的数是否大于右边的，如果是，则返回 true。     |
  | -lt    | 检测左边的数是否小于右边的，如果是，则返回 true。     |
  | -ge    | 检测左边的数是否大于等于右边的，如果是，则返回 true。 |
  | -le    | 检测左边的数是否小于等于右边的，如果是，则返回 true。 |

- 布尔运算符

  | 运算符 | 说明                                                | 举例                        |
  | :----- | :-------------------------------------------------- | :-------------------------- |
  | !      | 非运算，表达式为 true 则返回 false，否则返回 true。 | [ ! false ]                 |
  | -o     | 或运算，有一个表达式为 true 则返回 true。           | [ $a -lt 20 -o $b -gt 100 ] |
  | -a     | 与运算，两个表达式都为 true 才返回 true。           | [ $a -lt 20 -a $b -gt 100 ] |

- 逻辑运算符

  | 运算符 | 说明       | 举例                             |
  | :----- | :--------- | :------------------------------- |
  | &&     | 逻辑的 AND | [[ $a -lt 100 && $b -gt 100 ]]   |
  | \|\|   | 逻辑的 OR  | [[ $a -lt 100 \|\| $b -gt 100 ]] |

- 字符串运算符

  | 运算符 | 说明                                         | 举例         |
  | :----- | :------------------------------------------- | :----------- |
  | =      | 检测两个字符串是否相等，相等返回 true。      | [ $a = $b ]  |
  | !=     | 检测两个字符串是否不相等，不相等返回 true。  | [ $a != $b ] |
  | -z     | 检测字符串长度是否为0，为0返回 true。        | [ -z $a ]    |
  | -n     | 检测字符串长度是否不为 0，不为 0 返回 true。 | [ -n "$a" ]  |
  | $      | 检测字符串是否不为空，不为空返回 true。      | [ $a ]       |

- 文件测试运算符

  文件测试运算符用于检测 Unix 文件的各种属性：`file="/var/www/runoob/test.sh"`

  | 操作符  | 说明                                                         | 举例                      |
  | :------ | :----------------------------------------------------------- | :------------------------ |
  | -b file | 检测文件是否是块设备文件，如果是，则返回 true。              | [ -b $file ] 返回 false。 |
  | -c file | 检测文件是否是字符设备文件，如果是，则返回 true。            | [ -c $file ] 返回 false。 |
  | -d file | 检测文件是否是目录，如果是，则返回 true。                    | [ -d $file ] 返回 false。 |
  | -f file | 检测文件是否是普通文件（既不是目录，也不是设备文件），如果是，则返回 true。 | [ -f $file ] 返回 true。  |
  | -g file | 检测文件是否设置了 SGID 位，如果是，则返回 true。            | [ -g $file ] 返回 false。 |
  | -k file | 检测文件是否设置了粘着位(Sticky Bit)，如果是，则返回 true。  | [ -k $file ] 返回 false。 |
  | -p file | 检测文件是否是有名管道，如果是，则返回 true。                | [ -p $file ] 返回 false。 |
  | -u file | 检测文件是否设置了 SUID 位，如果是，则返回 true。            | [ -u $file ] 返回 false。 |
  | -r file | 检测文件是否可读，如果是，则返回 true。                      | [ -r $file ] 返回 true。  |
  | -w file | 检测文件是否可写，如果是，则返回 true。                      | [ -w $file ] 返回 true。  |
  | -x file | 检测文件是否可执行，如果是，则返回 true。                    | [ -x $file ] 返回 true。  |
  | -s file | 检测文件是否为空（文件大小是否大于0），不为空返回 true。     | [ -s $file ] 返回 true。  |
  | -e file | 检测文件（包括目录）是否存在，如果是，则返回 true。          | [ -e $file ] 返回 true。  |

### 3.2 if 条件语句介绍

```shell
# If条件判断语句，通常以if开头，fi结尾。也可加入else或者elif进行多条件的判断

# 单分支语句
if condition
then
    command1 
    command2
    ...
    commandN 
fi
# 写成一行：if [ $(ps -ef | grep -c "ssh") -gt 1 ]; then echo "true"; fi

# 双分支if 语句
if condition
then
    command1 
    command2
    ...
    commandN
else
    command
fi

# 多支条件语句 
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

示例：

```shell
a=10
b=20
# 条件表达式要放在方括号之间，并且要有空格
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

# 使用 ((...)) 作为判断语句，大于和小于可以直接使用 > 和 <
if (( $a == $b ))
then
   echo "a 等于 b"
elif (( $a > $b ))
then
   echo "a 大于 b"
elif (( $a < $b ))
then
   echo "a 小于 b"
else
   echo "没有符合的条件"
fi
```

### 3.3 case 选择语句介绍

`case ... esac` 为多选择语句，与其他语言中的 `switch ... case` 语句类似，是一种多分支选择结构。每个 case 分支用右圆括号开始，用两个分号 `;;` 表示 break，即执行结束，跳出整个 `case ... esac` 语句，esac（就是 case 反过来）作为结束标记。

```shell
# 取值后面必须为单词 in
case 值 in
模式1)
    command1
    command2
    ...
    commandN
    ;;
模式2)
    command1
    command2
    ...
    commandN
    ;;
esac
```

示例：

```shell
#!/bin/sh

site="runoob"

case "$site" in
   "runoob") echo "菜鸟教程"
   ;;
   "google") echo "Google 搜索"
   ;;
   "taobao") echo "淘宝网"
   ;;
   # 如果无一匹配模式，使用星号 * 捕获该值，再执行后面的命令
   *)  echo '未在范围的网站'
   ;;
esac
```

### 3.4 for 循环语句介绍

```shell
for var in item1 item2 ... itemN
do
    command1
    command2
    ...
    commandN
done

# 写成一行
for var in item1 item2 ... itemN; do command1; command2… done;


# 死循环
for (( ; ; ))
```

示例：

```shell
for loop in $(seq 1 254)
do
    echo "The value is: $loop"
done
```

### 3.5 while 循环语句介绍

```shell
while condition
do
    command
done

# 死循环
while : # while true 也可
do
    command
done
```

示例：

```shell
#!/bin/bash
int=1
while(( $int<=5 ))
do
    echo $int
    # 使用了 Bash let 命令，它用于执行一个或多个表达式，变量计算中不需要加上 $ 来表示变量
    let "int++"
done
```

### 3.6 until 循环语句介绍

until 循环执行一系列命令直至条件为 true 时停止，与 while 循环在处理方式上刚好相反

```shell
until condition
do
    command
done
```

示例：

```shell
#!/bin/bash

a=0

# 一般为条件表达式，如果返回值为 false，则继续执行循环体内的语句，否则跳出循环
until [ ! $a -lt 10 ]
do
   echo $a
   a=`expr $a + 1`
done
```

### 3.7 select 选择语句介绍

`select` 是一个类似于 for 循环的语句，一般用于选择

```shell
# 运行到 select 语句后，取值列表 value_list 中的内容会以菜单的形式显示出来，用户输入菜单编号，就表示选中了某个值，这个值就会赋给变量 variable，然后再执行循环体中的 statements
# 每次循环时 select 都会要求用户输入菜单编号，并使用环境变量 PS3 的值作为提示符，PS3 的默认值为#?，修改 PS3 的值就可以修改提示符
# select 是死循环，输入空值，或者输入的值无效，都不会结束循环，只有遇到 break 语句，或者按下 Ctrl+D 组合键才能结束循环
# 如果用户输入一个空值（什么也不输入，直接回车），会重新显示一遍菜单
select variable in value_list
do
    statements
done
```

示例：

```shell
#!/bin/bash

PS3="Please enter you select install menu:"

select i in http php mysql quit
do
	case $i in
        http)
        echo -e "\033[31m Test Httpd \033[0m" 
        ;;
        php)
        echo  -e "\033[32m Test PHP\033[0m"
        ;;
        mysql)
        echo -e "\033[33m Test MySQL.\033[0m"
        ;;
        quit)
        echo -e "\033[32m The System exit.\033[0m"
        exit
	esac
done
```

### 3.8 shell 函数介绍

Shell允许将一组命令集或语句形成一个可用块，这些块称为Shell函数，Shell函数的用于在于只需定义一次，后期随时使用即可，无需在Shell脚本中添加重复的语句块，其语法格式以`function name（）{`开头，以`}`结尾。

所有函数在使用前必须定义。这意味着必须将函数放在脚本开始部分，直至shell解释器首次发现它时，才可以使用

```shell
# 函数定义
# Shell 函数默认不能将参数传入（）内部
funWithParam(){
	# 在函数体内部，通过 $n 的形式来获取参数的值，例如，$1表示第一个参数，$2表示第二个参数
    echo "第一个参数为 $1 !"
    echo "第二个参数为 $2 !"
    # 注意，$10 不能获取第十个参数，获取第十个参数需要${10}。当n>=10时，需要使用${n}来获取参数
    echo "第十个参数为 $10 !"
    echo "第十个参数为 ${10} !"
    echo "第十一个参数为 ${11} !"
    echo "参数总数有 $# 个!"
    echo "作为一个字符串输出所有参数 $* !"
    # 结果返回，可以显示加：return 返回；如果不加，将以最后一条命令运行结果，作为返回值。 return后跟数值0-255
    return 10
}
# 直接通过函数名调用
# Shell函数参数传递，在调用函数名称传递
funWithParam 1 2 3 4 5 6 7 8 9 34 73

# 函数返回值在调用该函数后通过 $? 来获得
echo "函数返回结果为: $? "
```



## 四、shell 编程实战案例

### 4.1 shell 脚本实战之 系统备份脚本

- Tar 工具全备、增量备份网站，Shell 脚本实现自动打包备份

```shell
#!/bin/bash
#Auto Backup Linux System Files
#by author rivers on 2021-09-28

SOURCE_DIR=(
    $*
)
TARGET_DIR=/data/backup/
YEAR=`date +%Y`
MONTH=`date +%m`
DAY=`date +%d`
WEEK=`date +%u`
A_NAME=`date +%H%M`
FILES=system_backup.tgz
CODE=$?
if
    [ -z "$*" ]；then
    echo -e "\033[32mUsage:\nPlease Enter Your Backup Files or Directories\n--------------------------------------------\n\nUsage: { $0 /boot /etc}\033[0m"
    exit
fi
#Determine Whether the Target Directory Exists
if
    [ ! -d $TARGET_DIR/$YEAR/$MONTH/$DAY ]；then
    mkdir -p $TARGET_DIR/$YEAR/$MONTH/$DAY
    echo -e "\033[32mThe $TARGET_DIR Created Successfully !\033[0m"
fi
#EXEC Full_Backup Function Command
Full_Backup()
{
if
    [ "$WEEK" -eq "7" ]；then
    rm -rf $TARGET_DIR/snapshot
    cd $TARGET_DIR/$YEAR/$MONTH/$DAY ；tar -g $TARGET_DIR/snapshot -czvf $FILES ${SOURCE_DIR[@]}
    [ "$CODE" == "0" ]&&echo -e  "--------------------------------------------\n\033[32mThese Full_Backup System Files Backup Successfully !\033[0m"
fi
}
#Perform incremental BACKUP Function Command
Add_Backup()
{
   if
        [ $WEEK -ne "7" ]；then
        cd $TARGET_DIR/$YEAR/$MONTH/$DAY ；tar -g $TARGET_DIR/snapshot -czvf $A_NAME$FILES ${SOURCE_DIR[@]}
        [ "$CODE" == "0" ]&&echo -e  "-----------------------------------------\n\033[32mThese Add_Backup System Files $TARGET_DIR/$YEAR/$MONTH/$DAY/${YEAR}_$A_NAME$FILES Backup Successfully !\033[0m"
   fi
}
sleep 3 
Full_Backup；Add_Backup
```

### 4.2 shell 脚本 实战 之收集系统信息

- Shell 脚本实现服务器信息自动收集

```shell
cat <<EOF
++++++++++++++++++++++++++++++++++++++++++++++
++++++++Welcome to use system Collect+++++++++
++++++++++++++++++++++++++++++++++++++++++++++
EOF
ip_info=`ifconfig |grep "Bcast"|tail -1 |awk '{print $2}'|cut -d: -f 2`
cpu_info1=`cat /proc/cpuinfo |grep 'model name'|tail -1 |awk -F: '{print $2}'|sed 's/^ //g'|awk '{print $1,$3,$4,$NF}'`
cpu_info2=`cat /proc/cpuinfo |grep "physical id"|sort |uniq -c|wc -l`
serv_info=`hostname |tail -1`
disk_info=`fdisk -l|grep "Disk"|grep -v "identifier"|awk '{print $2,$3,$4}'|sed 's/,//g'`
mem_info=`free -m |grep "Mem"|awk '{print "Total",$1,$2"M"}'`
load_info=`uptime |awk '{print "Current Load: "$(NF-2)}'|sed 's/\,//g'`
mark_info='BeiJing_IDC'
echo -e "\033[32m-------------------------------------------\033[1m"
echo IPADDR:${ip_info}
echo HOSTNAME:$serv_info
echo CPU_INFO:${cpu_info1} X${cpu_info2}
echo DISK_INFO:$disk_info
echo MEM_INFO:$mem_info
echo LOAD_INFO:$load_info
echo -e "\033[32m-------------------------------------------\033[0m"
echo -e -n "\033[36mYou want to write the data to the databases? \033[1m" ；read ensure

if      [ "$ensure" == "yes" -o "$ensure" == "y" -o "$ensure" == "Y" ];then
        echo "--------------------------------------------"
        echo -e  '\033[31mmysql -uaudit -p123456 -D audit -e ''' "insert into audit_system values('','${ip_info}','$serv_info','${
cpu_info1} X${cpu_info2}','$disk_info','$mem_info','$load_info','$mark_info')" ''' \033[0m '
        mysql -uroot -p123456 -D test -e "insert into audit_system values('','${ip_info}','$serv_info','${cpu_info1} X${cpu_info2}
','$disk_info','$mem_info','$load_info','$mark_info')"
else
        echo "Please wait，exit......"
        exit
fi
```

### 4.3 shell 脚本实战 之 一键部署 lnmp 架构 

- 批量部署 lnmp 架构

```shell
[root@web-server01~/script]# vim lnmp.sh 
#!/bin/bash
#install lnmp
#by author rivers on 2021-9-28

# nginx 环境准备
Nginx_url=https://nginx.org/download/nginx-1.20.1.tar.gz
Nginx_prefix=/usr/local/nginx

# mysql 环境准备
Mysql_version=mysql-5.5.20.tar.gz
Mysql_dir=mysql-5.5.20
Mysql_url=https://downloads.mysql.com/archives/get/p/23/file/mysql-5.5.20.tar.gz
Mysql_prefix=/usr/local/mysql/

# php 环境准备
Php_version=php-7.2.10.tar.gz
Php_prefix=/usr/local/php-7.2.10/


function nginx_install(){

if [[ "$1" -eq "1" ]];then
        if [ $? -eq 0 ];then
                make && make install
        fi

fi
}



function mysql_install(){
if [[ "$1" -eq "2" ]];then
-DMYSQL_UNIX_ADDR=/tmp/mysql.sock \
-DMYSQL_DATADIR=/data/mysql \
-DSYSCONFDIR=/etc \
-DMYSQL_USER=mysql \
-DMYSQL_TCP_PORT=3306 \
-DWITH_XTRADB_STORAGE_ENGINE=1 \
-DWITH_INNOBASE_STORAGE_ENGINE=1 \
-DWITH_PARTITION_STORAGE_ENGINE=1 \
-DWITH_BLACKHOLE_STORAGE_ENGINE=1 \
-DWITH_MYISAM_STORAGE_ENGINE=1 \
-DWITH_READLINE=1 \
-DENABLED_LOCAL_INFILE=1 \
-DWITH_EXTRA_CHARSETS=1 \
-DDEFAULT_CHARSET=utf8 \
-DDEFAULT_COLLATION=utf8_general_ci \
-DEXTRA_CHARSETS=all \
                echo -e "\033[32mThe $Mysql_dir Server Install Success !\033[0m"
        else
                echo -e "\033[32mThe $Mysql_dir Make or Make install ERROR,Please Check......"
                exit 0
fi
/bin/cp support-files/my-small.cnf  /etc/my.cnf
/bin/cp support-files/mysql.server /etc/init.d/mysqld
chmod +x /etc/init.d/mysqld
chkconfig --add mysqld
chkconfig mysqld on
fi
}


function php_install(){
if [[ "$1" -eq "3" ]];then
        if [ $? -eq 0 ];then
                make ZEND_EXTRA_LIBS='-liconv' && make install
if [[ "$1" -eq "3" ]];then
 wget $Php_url && tar xf $Php_version && cd $Php_dir && yum install   bxml2* bzip2* libcurl*  libjpeg* libpng* freetype* gmp* libm
crypt* readline* libxslt* -y && ./configure --prefix=$Php_prefix --disable-fileinfo --enable-fpm --with-config-file-path=/etc --wi
  -config-file-scan-dir=/etc/php.d --with-openssl --with-zlib --with-curl --enable-ftp --with-gd --with-xmlrpc --with-jpeg-dir --w
ith-png-dir --with-freetype-dir --enable-gd-native-ttf --enable-mbstring --with-mcrypt=/usr/local/libmcrypt --enable-zip --enable-
mysqlnd --with-mysqli=mysqlnd --with-pdo-mysql=mysqlnd --with-mysql-sock=/var/lib/mysql/mysql.sock --without-pear --enable-bcmath
        if [ $? -eq 0 ];then
                make ZEND_EXTRA_LIBS='-liconv' && make install
                echo -e "\n\033[32m-----------------------------------------------\033[0m"
                echo -e "\033[32mThe $Php_version Server Install Success !\033[0m"
        else
                echo -e "\033[32mThe $Php_version Make or Make install ERROR,Please Check......"
                exit 0
        fi
fi
}


PS3="Please enter you select install menu:"
select i in nginx mysql php quit
do
"lnmp.sh" 113L, 3516C written                                                                                   
[root@web-server01~/script]# vim lnmp.sh 
chkconfig --add mysqld
chkconfig mysqld on
fi
}


function php_install(){
if [[ "$1" -eq "3" ]];then
        if [ $? -eq 0 ];then
                make ZEND_EXTRA_LIBS='-liconv' && make install
                echo -e "\n\033[32m-----------------------------------------------\033[0m"
                echo -e "\033[32mThe $Php_version Server Install Success !\033[0m"
        else
                echo -e "\033[32mThe $Php_version Make or Make install ERROR,Please Check......"
                exit 0
        fi
fi
}


PS3="Please enter you select install menu:"
select i in nginx mysql php quit
do

case $i in
        nginx)
        nginx_install 1
        ;;
        mysql)
        mysql_install 2
        ;;
        php)
        php_install 3
        ;;
        quit)
        exit
esac
done
```


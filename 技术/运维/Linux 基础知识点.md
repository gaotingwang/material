## Linux 系统种类

- 红帽企业版 `Linux` ： `RHEL` 是全世界内使用最广泛的 `Linux` 系统。它具有极强的性能与稳定性，是众多生成环境中使用的（收费的）系统。
- `Fedora` ：由红帽公司发布的桌面版系统套件，用户可以免费体验到最新的技术或工具，这些技术或工具在成熟后会被加入到 `RHEL` 系统中，因此 `Fedora` 也成为 `RHEL` 系统的试验版本。
- `CentOS` ：通过把 `RHEL` 系统重新编译并发布给用户免费使用的 `Linux` 系统，具有广泛的使用人群。
- `Deepin` ：中国发行，对优秀的开源成品进行集成和配置。
- `Debian` ：稳定性、安全性强，提供了免费的基础支持，在国外拥有很高的认可度和使用率。
- `Ubuntu` ：是一款派生自 `Debian` 的操作系统，对新款硬件具有极强的兼容能力。 `Ubuntu` 与 `Fedora` 都是极其出色的 `Linux` 桌面系统，而且 `Ubuntu` 也可用于服务器领域。

## 命令

### 命令行提示符

进入命令行环境以后，用户会看到 `Shell` 的提示符。提示符往往是一串前缀，最后以一个美元符号 `$` 结尾，用户可以在这个符号后面输入各种命令。

执行一个简单的命令 `pwd` ：

```shell
[root@iZm5e8dsxce9ufaic7hi3uZ ~]# pwd
/root
```

命令解析：

- `root`：表示用户名；
- `iZm5e8dsxce9ufaic7hi3uZ`：表示主机名；
- `~`：表示目前所在目录为家目录，其中 `root` 用户的家目录是 `/root` ，普通用户的家目录在 `/home` 下；
- `#`：指示所具有的权限（ `root` 用户为 `#` ，普通用户为 `$` ）。
- 执行 `whoami` 命令可以查看当前用户名；
- 执行 `hostname` 命令可以查看当前主机名；

[备注] `root` 是超级用户，具备操作系统的一切权限。

### 命令格式

```shell
command parameters（命令 参数）
```

#### 长短参数

```
单个参数：ls -a（a 是英文 all 的缩写，表示“全部”）
多个参数：ls -al（全部文件 + 列表形式展示）
单个长参数：ls --all
多个长参数：ls --reverse --all
长短混合参数：ls --all -l
```

#### 参数值

```
短参数：command -p 10（例如：ssh root@121.42.11.34 -p 22）
长参数：command --paramters=10（例如：ssh root@121.42.11.34 --port=22）
```

## 快捷方式

在学习 `Linux` 命令之前，有一些快捷方式，是必须要提前掌握的，它将贯穿整个 `Linux` 使用生涯。

- 通过上下方向键 ↑ ↓ 来调取过往执行过的 `Linux` 命令；
- 命令或参数仅需输入前几位就可以用 `Tab` 键补全；
- `Ctrl + R` ：用于查找使用过的命令（`history` 命令用于列出之前使用过的所有命令，然后输入 `!` 命令加上编号( `!2` )就可以直接执行该历史命令）；
- `Ctrl + L`：清除屏幕并将当前行移到页面顶部；
- `Ctrl + C`：中止当前正在执行的命令；
- `Ctrl + U`：从光标位置剪切到行首；
- `Ctrl + K`：从光标位置剪切到行尾；
- `Ctrl + W`：剪切光标左侧的一个单词；
- `Ctrl + Y`：粘贴 `Ctrl + U | K | W` 剪切的命令；
- `Ctrl + A`：光标跳到命令行的开头；
- `Ctrl + E`：光标跳到命令行的结尾；
- `Ctrl + D`：关闭 `Shell` 会话；

## 文件和目录

### 文件组织

<img src="https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/linux%E7%9B%AE%E5%BD%95%E7%BB%93%E6%9E%84.jpg" alt="Linux目录结构" style="zoom:50%;" />

```shell
.                            # 根目录，有且只有一个
|-- bin -> usr/bin           # bin 是 Binaries (二进制文件) 的缩写, 包含了会被所有用户使用的可执行程序
|-- boot                     # 包含与 Linux 启动密切相关的文件，包括连接文件以及镜像文件
|-- dev                      # dev 是 Device(设备) 的缩写, 该目录下存放的是 Linux 的外部设备，在 Linux 中访问设备的方式和访问文件的方式是相同的
|-- etc                      # etc 是 Etcetera(等等) 的缩写，这个目录用来存放系统的配置文件
|-- home                     # 用户私人目录，在 Linux 中，每个用户都有一个自己的目录，一般该目录名是以用户的账号命名的
|   |-- alice                       # 代表不同用户
|   |-- bob
|   |-- eve
|-- root                     # 该目录为系统管理员，也称作超级权限者的用户主目录
|-- lib                      # Library(库) 的缩写，包含被程序所调用的库文件，几乎所有的应用程序都需要用到这些共享库
|-- sbin -> usr/sbin         # Superuser Binaries (超级用户的二进制文件) 的缩写，这里存放的是系统管理员使用的系统管理程序
|-- opt                      # optional 缩写，是给主机安装额外软件所摆放的目录，用于安装第三方软件和插件
|-- tmp                      # temporary(临时) 的缩写，普通用户和临时程序存放临时文件的地方
|-- usr                      # unix shared resources(共享资源) 的缩写，这是一个非常重要的目录，用户的很多应用程序和文件都放在这个目录下
|   |-- bin                         # 系统用户使用的应用程序
|   |-- src                         # 内核源代码默认的放置目录
|   |-- sbin                        # 超级用户使用的比较高级的管理程序和系统守护程序
|   |-- tmp
|-- media                    # linux 系统会自动识别一些设备，例如U盘、光驱等等，当识别后，Linux 会把识别的设备挂载到这个目录下
|-- mnt                      # mount缩写，该目录是为了让用户临时挂载别的文件系统，类似media，可以将光驱挂载在/mnt/上，进入该目录可以查看光驱中内容
|-- srv                      # 该目录存放一些服务启动之后需要提取的数据
|-- sys                      # 该目录下安装了 2.6 内核中新出现的一个文件系统 sysfs
|-- run                      # 是一个临时文件系统，存储系统启动以来的信息。当系统重启时，这个目录下的文件应该被删掉或清除
`-- var                      # variable(变量) 的缩写，这个目录中存放着在不断扩充着的东西，习惯将那些经常被修改的目录放在这个目录下，包括各种日志文件
    `-- tmp
```

在 Linux 系统中，有几个目录是比较重要的，平时需要注意不要误删除或者随意更改内部文件：

- `/etc`： 上边也提到了，这个是系统中的配置文件，如果更改了该目录下的某个文件可能会导致系统不能启动。

- `/bin, /sbin, /usr/bin, /usr/sbin`: 这是系统预设的执行文件的放置目录，比如 `ls` 就是在 `/bin/ls` 目录下的。

  值得提出的是 `/bin`、`/usr/bin` 是给系统用户使用的指令（除 root 外的通用用户），而`/sbin, /usr/sbin` 则是给 root 使用的指令。

- `/var`： 这是一个非常重要的目录，系统上跑了很多程序，那么每个程序都会有相应的日志产生，而这些日志就被记录到这个目录下，具体在 `/var/log` 目录下，另外 mail 的预设放置也是在这里。

### 查看路径

#### pwd

显示当前目录的路径：

```shell
$ pwd
/tmp
```

#### which

查看命令的可执行文件所在路径， `Linux` 下，每一条命令其实都对应一个可执行程序，在终端中输入命令，按回车的时候，就是执行了对应的那个程序， `which` 命令本身对应的程序也存在于 `Linux` 中。总的来说一个命令就是一个可执行程序：

```shell
$ which ls
/usr/bin/ls
```

### 目录浏览和切换

#### ls

列出文件和目录，它是 `Linux` 最常用的命令之一。

【常用参数】

- `-a` 显示所有文件和目录包括隐藏的
- `-l` 显示详细列表
- `-h` 适合人类阅读的
- `-t` 按文件最近一次修改时间排序
- `-i` 显示文件的 `inode` （ `inode` 是文件内容的标识）

```shell
$ ls -alh
total 16K
drwxrwxrwt 1 root root 4.0K Feb 27 06:50 .
drwxr-xr-x 1 root root 4.0K Feb 22 09:51 ..
-rw-r--r-x 1 root root  358 Feb 23 09:47 file.sh
-rw-r--r-x 1 root root  324 Feb 22 09:39 helloworld.sh
```

#### cd

`cd` 是英语 `change directory` 的缩写，表示切换目录。

```shell
cd /             --> 跳转到根目录
cd ~             --> 跳转到家目录
cd ..            --> 跳转到上级目录
cd ./home        --> 跳转到当前目录的home目录下
cd /home/lion    --> 跳转到根目录下的home目录下的lion目录
cd               --> 不添加任何参数，也是回到家目录
```

[注意] 输入`cd /ho` + 单次 `tab` 键会自动补全路径 + 两次 `tab` 键会列出所有可能的目录列表。

#### du

列举目录大小信息。

【常用参数】

- `-h` 适合人类阅读的；
- `-a` 同时列举出目录下文件的大小信息；
- `-s` 只显示总计大小，不显示具体信息。

### 文件浏览和创建

#### cat

一次性显示文件所有内容，更适合查看小的文件。

```shell
$ cat cloud-init.log
```

【常用参数】

- `-n` 显示行号。

#### less

分页显示文件内容，更适合查看大的文件。

```shell
$ less cloud-init.log
```

【快捷操作】

- 空格键：前进一页（一个屏幕）；
- `b` 键：后退一页；
- 回车键：前进一行；
- `y` 键：后退一行；
- 上下键：回退或前进一行；
- `d` 键：前进半页；
- `u` 键：后退半页；
- `q` 键：停止读取文件，中止 `less` 命令；
- `=` 键：显示当前页面的内容是文件中的第几行到第几行以及一些其它关于本页内容的详细信息；
- `h` 键：显示帮助文档；
- `/` 键：进入搜索模式后，按 `n` 键跳到一个符合项目，按 `N` 键跳到上一个符合项目，同时也可以输入正则表达式匹配。

#### head

显示文件的开头几行（默认是10行）

```shell
$ head cloud-init.log
```

【参数】

- `-n` 指定行数 `head cloud-init.log -n 2`

#### tail

显示文件的结尾几行（默认是10行）

```shell
$ tail cloud-init.log
```

【参数】

- `-n` 指定行数 `tail cloud-init.log -n 2`
- `-f` 会每过1秒检查下文件是否有更新内容，也可以用 `-s` 参数指定间隔时间 `tail -f -s 4 xxx.log`

#### touch

创建一个文件

```shell
$ touch new_file
```

#### mkdir

创建一个目录

```shell
$ mkdir new_folder
```

【常用参数】

- `-p` 递归的创建目录结构 `mkdir -p one/two/three`

### 文件的复制和移动

#### cp

拷贝文件和目录

```shell
cp file file_copy       --> file 是目标文件，file_copy 是拷贝出来的文件
cp file one             --> 把 file 文件拷贝到 one 目录下，并且文件名依然为 file
cp file one/file_copy   --> 把 file 文件拷贝到 one 目录下，文件名为file_copy
cp *.txt folder         --> 把当前目录下所有 txt 文件拷贝到 folder 目录下
```

【常用参数】

- `-r` 递归的拷贝，常用来拷贝一整个目录

#### mv

移动（重命名）文件或目录，与cp命令用法相似。

```shell
mv file one         --> 将 file 文件移动到 one 目录下
mv new_folder one   --> 将 new_folder 文件夹移动到one目录下
mv *.txt folder     --> 把当前目录下所有 txt 文件移动到 folder 目录下
mv file new_file    --> file 文件重命名为 new_file
```

### 文件的删除和链接

#### rm

删除文件和目录，由于 `Linux` 下没有回收站，一旦删除非常难恢复，因此需要谨慎操作

```shell
rm new_file  --> 删除 new_file 文件
rm f1 f2 f3  --> 同时删除 f1 f2 f3 3个文件
```

【常用参数】

- `-i` 向用户确认是否删除；
- `-f` 文件强制删除；
- `-r` 递归删除文件夹，著名的删除操作 `rm -rf` 。

#### ln

英文 `Link` 的缩写，表示创建链接。学习创建链接之前，首先要理解链接是什么：

先来看看 `Linux` 的文件是如何存储的：`Linux` 文件的存储方式分为3个部分，文件名、文件内容以及权限，其中文件名的列表是存储在硬盘的其它地方和文件内容是分开存放的，每个文件名通过 `inode` 标识绑定到文件内容。

Linux 下有两种链接类型：硬链接和软链接。

##### 硬链接

使链接的两个文件共享同样文件内容，就是同样的 `inode` ，一旦文件1和文件2之间有了硬链接，那么修改任何一个文件，修改的都是同一块内容。
它的缺点是，只能创建指向文件的硬链接，不能创建指向目录的（其实也可以，但比较复杂）。而软链接都可以，因此软链接使用更加广泛。

```shell
$ ln file1 file2      --> 创建 file2 为 file1 的硬链接
```

![硬链接](https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/%E7%A1%AC%E9%93%BE%E6%8E%A5.jpg)

如果用 `rm file1` 来删除 `file1` ，对 `file2` 没有什么影响，对于硬链接来说，删除任意一方的文件，共同指向的文件内容并不会从硬盘上删除。只有同时删除了 `file1` 与 `file2` 后，它们共同指向的文件内容才会消失

##### 软链接

软链接就类似 `windows` 下快捷方式。

```shell
$ ln -s file1 file2
```

![软链接](https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/%E8%BD%AF%E9%93%BE%E6%8E%A5.jpg)

其实 `file2` 只是 `file1` 的一个快捷方式，它指向的是 `file1` ，所以显示的是 `file1` 的内容，但其实 `file2` 的 `inode` 与 `file1` 并不相同。如果删除了 `file2` 的话， `file1` 是不会受影响的，但如果删除 `file1` 的话， `file2` 就会变成死链接，因为指向的文件不见了。

### 文件查找

#### locate

搜索包含关键字的所有文件和目录。后接需要查找的文件名，也可以用正则表达式。

##### 安装 locate

```shell
$ yum -y install mlocate       --> 安装包
$ updatedb                     --> 更新数据库
$ locate file.txt
$ locate fil*.txt
```

[注意] `locate` 命令会去文件数据库中查找命令，而不是全磁盘查找，因此刚创建的文件并不会更新到数据库中，所以无法被查找到，可以执行 `updatedb` 命令去更新数据库。

#### find

用于查找文件，它会去遍历你的实际硬盘进行查找，而且它允许我们对每个找到的文件进行后续操作，功能非常强大。

```shell
$ find <何处> <何物> <做什么>
```

- 何处：指定在哪个目录查找，此目录的所有子目录也会被查找。
- 何物：查找什么，可以根据文件的名字来查找，也可以根据其大小来查找，还可以根据其最近访问时间来查找。
- 做什么：找到文件后，可以进行后续处理，如果不指定这个参数， `find` 命令只会显示找到的文件。

##### 根据文件名查找

```shell
find -name "file.txt"             --> 当前目录以及子目录下通过名称查找文件
find . -name "syslog"             --> 当前目录以及子目录下通过名称查找文件
find / -name "syslog"             --> 整个硬盘下查找syslog
find /var/log -name "syslog"      --> 在指定的目录/var/log下查找syslog文件
find /var/log -name "syslog*"     --> 查找syslog1、syslog2 ... 等文件，通配符表示所有
find /var/log -name "*syslog*"    --> 查找包含syslog的文件 
```

[注意] `find` 命令只会查找完全符合 “何物” 字符串的文件，而 `locate` 会查找所有包含关键字的文件。

##### 根据文件大小查找

```shell
find /var -size +10M              --> /var 目录下查找文件大小超过 10M 的文件
find /var -size -50k              --> /var 目录下查找文件大小小于 50k 的文件
find /var -size +1G               --> /var 目录下查找文件大小查过 1G 的文件
find /var -size 1M                --> /var 目录下查找文件大小等于 1M 的文件
```

##### 根据文件最近访问时间查找

```shell
find -name "*.txt" -atime -7      --> 近 7天内访问过的.txt结尾的文件
```

##### 仅查找目录或文件

```shell
find . -name "file" -type f       --> 只查找当前目录下的file文件
find . -name "file" -type d       --> 只查找当前目录下的file目录
```

##### 操作查找结果

```shell
find -name "*.txt" -printf "%p - %u\n"    --> 找出所有后缀为txt的文件，并按照 %p - %u\n 格式打印，其中%p=文件名，%u=文件所有者
find -name "*.jpg" -delete                --> 删除当前目录以及子目录下所有.jpg为后缀的文件，不会有删除提示，因此要慎用
find -name "*.c" -exec chmod 600 {} \;    --> 对每个.c结尾的文件，都进行 -exec 参数指定的操作，{} 会被查找到的文件替代，\; 是必须的结尾
find -name "*.c" -ok chmod 600 {} \;      --> 和上面的功能一致，会多一个确认提示
```



## 用户与权限

### 用户操作

`Linux` 是一个多用户的操作系统。在 `Linux` 中，理论上来说，我们可以创建无数个用户，但是这些用户是被划分到不同的群组里面的，有一个用户，名叫 `root` ，是一个很特殊的用户，它是超级用户，拥有最高权限。

自己创建的用户是有限权限的用户，这样大大提高了 `Linux` 系统的安全性，有效防止误操作或是病毒攻击，但是我们执行的某些命令需要更高权限时可以使用 `sudo` 命令。

#### sudo

以 `root` 身份运行命令

```shell
$ sudo date            --> 当然查看日期是不需要sudo的这里只是演示，sudo 完之后一般还需要输入用户密码的
```

#### useradd + passwd

- `useradd` 添加新用户

   `-r` 建立系统帐号，`-g <群组>` 指定用户所属的群组，`-m` 制定用户的登入目录。

- `passwd` 修改用户密码

这两个命令需要 `root` 用户权限

使用 `useradd` 指令所建立的帐号，实际上是保存在 `/etc/passwd` 文本文件中

```shell
$ useradd lion         --> 添加一个lion用户，添加完之后在 /home 路径下可以查看
$ passwd lion          --> 修改lion用户的密码
```

#### userdel

删除用户，需要 `root` 用户权限

```shell
$ userdel lion          --> 只会删除用户名，不会从/home中删除对应文件夹
$ userdel -r lion       --> 会同时删除/home下的对应文件夹
```

#### su

切换用户，需要 `root` 用户权限

```shell
$ sudo su               --> 切换为root用户（exit 命令或 CTRL + D 快捷键都可以使普通用户切换为 root 用户）
$ su lion               --> 切换为普通用户
$ su -                  --> 切换为root用户
```

### 群组的管理

`Linux` 中每个用户都属于一个特定的群组，如果你不设置用户的群组，默认会创建一个和它的用户名一样的群组，并且把用户划归到这个群组。

#### groupadd

创建群组，用法和 `useradd` 类似。

```shell
$ groupadd friends
```

#### groupdel

删除一个已存在的群组

```shell
$ groupdel foo                 --> 删除foo群组
```

#### groups

查看用户所在群组

```shell
$ useradd -g friends -r lion   --> 创建lion用户，将其加到friends用户组
$ groups lion                  --> 查看 lion 用户所在的群组
```

#### usermod

用于修改用户的账户。

【常用参数】

- `-l`： 对用户重命名。需要注意的是 `/home` 中的用户家目录的名字不会改变，需要手动修改。
- `-g`： 修改用户所在的群组，例如 `usermod -g friends lion` 修改 `lion` 用户的群组为 `friends` 。
- `-G`： 一次性让用户添加多个群组，例如 `usermod -G friends,foo,bar lion` 。
- `-a`： `-G` 会让你离开原先的群组，如果你不想这样做的话，就得再添加 `-a` 参数，意味着 `append` 追加的意思。

#### chgrp

用于修改文件的群组。

```shell
$ chgrp [–R] bar file.txt           --> file.txt文件的群组修改为bar
```

#### chown

改变文件的所有者，需要 `root` 身份才能运行。

```Shell
$ chown lion file.txt          --> 把其它用户创建的file.txt转让给lion用户
$ chown lion:bar file.txt      --> 把file.txt的用户改为lion，群组改为bar
```

【常用参数】

- `-R` 递归设置子目录和子文件， `chown -R lion:lion /home/frank` 把 `frank` 文件夹的用户和群组都改为 `lion` 。

### 文件权限管理

#### chmod

修改访问权限。

```shell
$ chmod 740 file.txt
```

【常用参数】

- `-R` 可以递归地修改文件访问权限，例如 `chmod -R 777 /home/lion`

修改权限的确简单，但是理解其深层次的意义才是更加重要的。下面系统的学习 `Linux` 的文件权限：

```shell
[root@lion ~]# ls -l
drwxr-xr-x 5 root root 4096 Apr 13  2020 climb
lrwxrwxrwx 1 root root    7 Jan 14 06:41 hello2.c -> hello.c
-rw-r--r-- 1 root root  149 Jan 13 06:14 hello.c
```

在 Linux 中第一个字符代表这个文件是目录、文件或链接文件等等。

- 当为 `d` 则是目录
- 当为 `-` 则是文件；
- 若是 `l` 则表示为链接文档(link file)；
- 若是 `b` 则表示为装置文件里面的可供储存的接口设备(可随机存取装置)；
- 若是 `c` 则表示为装置文件里面的串行端口设备，例如键盘、鼠标(一次性读取装置)。

第 **0** 位确定文件类型，第 **1-3** 位确定属主（该文件的所有者）拥有该文件的权限；第4-6位确定属组（所有者的同组用户）拥有该文件的权限；第7-9位确定其他用户拥有该文件的权限。

接下来的字符中，以三个为一组，且均为 `rwx` 的三个参数的组合。其中， `r` 代表可读(read)、 `w` 代表可写(write)、 `x` 代表可执行(execute)。 要注意的是，这三个权限的位置不会改变，如果没有权限，就会出现减号 `-` 。

现在再来理解这句权限 `drwxr-xr-x` 的意思：

- 它是一个文件夹；
- 它的所有者具有：读、写、执行权限；
- 它的群组用户具有：读、执行的权限，没有写的权限；
- 它的其它用户具有：读、执行的权限，没有写的权限。

`chmod` 它是不需要 `root` 用户才能运行的，只要你是此文件所有者，就可以用 `chmod` 来修改文件的访问权限。

##### 数字分配权限

可以使用数字来代表各个权限，各权限的分数对照如下：

- `r`：4
- `w`：2
- `x`：1

```shell
$ chmod 640 hello.c 

# 分析
6 = 4 + 2 + 0 表示所有者具有 rw 权限
4 = 4 + 0 + 0 表示群组用户具有 r 权限
0 = 0 + 0 + 0 表示其它用户没有权限

对应文字权限为：-rw-r-----
```

##### 用字母来分配权限

- `u` ： `user` 的缩写，用户的意思，表示所有者。
- `g` ： `group` 的缩写，群组的意思，表示群组用户。
- `o` ： `other` 的缩写，其它的意思，表示其它用户。
- `a` ： `all` 的缩写，所有的意思，表示所有用户。
- `+` ：加号，表示添加权限。
- `-` ：减号，表示去除权限。
- `=` ：等于号，表示分配权限。

```shell
chmod u+rx file               --> 文件file的所有者增加读和运行的权限
chmod g+r file                --> 文件file的群组用户增加读的权限
chmod o-r file                --> 文件file的其它用户移除读的权限
chmod g+r o-r file            --> 文件file的群组用户增加读的权限，其它用户移除读的权限
chmod go-r file               --> 文件file的群组和其他用户移除读的权限
chmod +x file                 --> 文件file的所有用户增加运行的权限
chmod u=rwx,g=r,o=- file      --> 文件file的所有者分配读写和执行的权限，群组其它用户分配读的权限，其他用户没有任何权限
```



## 软件仓库

`Linux` 下软件是以包的形式存在，一个软件包其实就是软件的所有文件的压缩包，是二进制的形式，包含了安装软件的所有指令。 `Red Hat` 家族的软件包后缀名一般为 `.rpm` ， `Debian` 家族的软件包后缀是 `.deb` 。

`Linux` 的包都存在一个仓库，叫做软件仓库，它可以使用 `yum` 来管理软件包， `yum` 是 `CentOS` 中默认的包管理工具，适用于 `Red Hat` 一族。可以理解成 `Node.js` 的 `npm` 。

### yum 常用命令

- `yum update | yum upgrade` 更新软件包
- `yum search xxx` 搜索相应的软件包
- `yum install xxx` 安装软件包
- `yum remove xxx` 删除软件包

### 切换 CentOS 软件源

有时候 `CentOS` 默认的 `yum` 源不一定是国内镜像，导致 `yum` 在线安装及更新速度不是很理想。这时候需要将 `yum` 源设置为国内镜像站点。国内主要开源的镜像站点是网易和阿里云。

1、首先备份系统自带 `yum` 源配置文件 `mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup`

2、下载阿里云的 `yum` 源配置文件到 `/etc/yum.repos.d/CentOS7`

```shell
$ wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
```

3、生成缓存

```shell
$ yum makecache
```



## 阅读手册

`Linux` 命令种类繁杂，凭借记忆不可能全部记住，因此学会查用手册是非常重要的。

### man

#### 安装更新 man

```shell
sudo yum install -y man-pages     --> 安装
sudo mandb                        --> 更新
```

#### man 手册种类

1. 可执行程序或 `Shell` 命令；
2. 系统调用（ `Linux` 内核提供的函数）；
3. 库调用（程序库中的函数）；
4. 文件（例如 `/etc/passwd` ）；
5. 特殊文件（通常在 `/dev` 下）；
6. 游戏；
7. 杂项（ `man(7)` ，`groff(7)` ）；
8. 系统管理命令（通常只能被 `root` 用户使用）；
9. 内核子程序。

#### man + 数字 + 命令

输入 man + 数字 + 命令/函数，可以查到相关的命令和函数，若不加数字， `man` 默认从数字较小的手册中寻找相关命令和函数

```shell
$ man 3 rand               --> 表示在手册的第三部分查找 rand 函数
$ man ls                   --> 查找 ls 用法手册
```

man 手册核心区域解析：(以 `man pwd` 为例)

```shell
NAME # 命令名称和简单描述
     pwd -- return working directory name

SYNOPSIS # 使用此命令的所有方法
     pwd [-L | -P]

DESCRIPTION # 包括所有参数以及用法
     The pwd utility writes the absolute pathname of the current working directory to the standard output.

     Some shells may provide a builtin pwd command which is similar or identical to this utility.  Consult the builtin(1) manual page.

     The options are as follows:

     -L      Display the logical current working directory.

     -P      Display the physical current working directory (all symbolic links resolved).

     If no options are specified, the -L option is assumed.

SEE ALSO # 扩展阅读相关命令
     builtin(1), cd(1), csh(1), sh(1), getcwd(3)
```

### help

`man` 命令像新华词典一样可以查询到命令或函数的详细信息，但其实我们还有更加快捷的方式去查询， `command \--help` 或 `command \-h` ，它没有 `man` 命令显示的那么详细，但是它更加易于阅读。
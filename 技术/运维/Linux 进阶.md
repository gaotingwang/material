## 文本操作

### grep

全局搜索一个正则表达式，并且打印到屏幕。简单来说就是，在文件中查找关键字，并显示关键字所在行。

**基础语法**：

```shell
$ grep text file # text代表要搜索的文本，file代表供搜索的文件

# 实例
[root@lion ~]# grep path /etc/profile
pathmunge () {
    pathmunge /usr/sbin
    pathmunge /usr/local/sbin
    pathmunge /usr/local/sbin after
    pathmunge /usr/sbin after
unset -f pathmunge
```

**常用参数**：

- `-A<显示行数> 或 --after-context=<显示行数>` : 除了显示符合范本样式的那一列之外，并显示该行之后的内容
- `-B<显示行数> 或 --before-context=<显示行数>`: 除了显示符合样式的那一行之外，并显示该行之前的内容
- `-C<显示行数> 或 --context=<显示行数>或-<显示行数>` : 除了显示符合样式的那一行之外，并显示该行之前后的内容。
- `-i` 忽略大小写， `grep -i path /etc/profile`
- `-n` 显示行号，`grep -n path /etc/profile`
- `-v` 只显示搜索文本不在的那些行，`grep -v path /etc/profile`
- `-r` 递归查找， `grep -r hello /etc` ，Linux 中还有一个 rgrep 命令，作用相当于 `grep -r`

**高级用法**：

`grep` 可以配合正则表达式使用。

```shell
grep -E path /etc/profile        --> 完全匹配path
grep -E ^path /etc/profile       --> 匹配path开头的字符串
grep -E [Pp]ath /etc/profile     --> 匹配path或Path
```

### sort

对文件的行进行排序。

**基础语法**：

```shell
sort name.txt # 对name.txt文件进行排序
```

**实例用法**：

为了演示方便，我们首先创建一个文件 `name.txt` ，放入以下内容：

```
Christopher
Shawn
Ted
Rock
Noah
Zachary
Bella
```

执行 `sort name.txt` 命令，会对文本内容进行排序。

**常用参数**：

- `-o` 将排序后的文件写入新文件， `sort -o name_sorted.txt name.txt` ；
- `-r` 倒序排序， `sort -r name.txt` ；
- `-R` 随机排序， `sort -R name.txt` ；
- `-n` 对数字进行排序，默认是把数字识别成字符串的，因此 138 会排在 25 前面，如果添加了 `-n` 数字排序的话，则 25 会在 138 前面。

### wc

`word count` 的缩写，用于文件的统计。它可以统计单词数目、行数、字符数，字节数等。

**基础语法**：

```shell
wc name.txt # 统计name.txt
```

**实例用法**：

```shell
[root@lion ~]# wc name.txt 
13 13 91 name.txt    # 第一个13，表示行数;第二个13，表示单词数;第三个91，表示字节数
```

**常用参数**：

- `-l` 只统计行数， `wc -l name.txt` ；
- `-w` 只统计单词数， `wc -w name.txt` ；
- `-c` 只统计字节数， `wc -c name.txt` ；
- `-m` 只统计字符数， `wc -m name.txt` 。

### uniq

删除文件中的重复内容。

基础语法

```shell
uniq name.txt # 去除name.txt重复的行数，并打印到屏幕上
uniq name.txt uniq_name.txt # 把去除重复后的文件保存为 uniq_name.txt
```

【注意】它只能去除连续重复的行数。

常用参数

- `-c` 统计重复行数， `uniq -c name.txt` ；
- `-d` 只显示重复的行数， `uniq -d name.txt` 。

### cut

剪切文件的一部分内容。

**基础语法**：

```shell
cut -c 2-4 name.txt # 剪切每一行第二到第四个字符
```

**常用参数**：

- `-d` 用于指定用什么分隔符（比如逗号、分号、双引号等等） `cut -d , name.txt` ；
- `-f` 表示剪切下用分隔符分割的哪一块或哪几块区域， `cut -d , -f 1 name.txt` 。



## 重定向管道流

在 `Linux` 中一个命令的去向可以有3个地方：终端、文件、作为另外一个命令的入参。

命令一般都是通过键盘输入，然后输出到终端、文件等地方，它的标准用语是 `stdin` 、 `stdout` 以及 `stderr` 。

- 标准输入 `stdin` ，终端接收键盘输入的命令，会产生两种输出；
- 标准输出 `stdout` ，终端输出的信息（不包含错误信息）；
- 标准错误输出 `stderr` ，终端输出的错误信息。

### 重定向

把本来要显示在终端的命令结果，输送到别的地方（到文件中或者作为其他命令的输入）。

**输出重定向 `>`**

`>` 表示重定向到新的文件， `cut -d , -f 1 notes.csv > name.csv` ，它表示通过逗号剪切 `notes.csv` 文件（剪切完有3个部分）获取第一个部分，重定向到 `name.csv` 文件。

看一个具体示例，学习它的使用，假设有一个文件 `notes.csv` ，文件内容如下：

```
Mark1,951/100,很不错1
Mark2,952/100,很不错2
Mark3,953/100,很不错3
Mark4,954/100,很不错4
Mark5,955/100,很不错5
Mark6,956/100,很不错6
```

执行命令： `cut -d , -f 1 notes.csv > name.csv` 最后输出如下内容：

```
Mark1
Mark2
Mark3
Mark4
Mark5
Mark6
```

【注意】使用 `>` 要注意，如果输出的文件不存在它会新建一个，如果输出的文件已经存在，则会覆盖。因此执行这个操作要非常小心，以免覆盖其它重要文件。

**输出重定向 `>>`**

表示重定向到文件末尾，因此它不会像 `>` 命令这么危险，它是追加到文件的末尾（当然如果文件不存在，也会被创建）。

再次执行 `cut -d , -f 1 notes.csv >> name.csv` ，则会把名字追加到 `name.csv` 里面。

```shell
Mark1
Mark2
Mark3
Mark4
Mark5
Mark6
Mark1
Mark2
Mark3
Mark4
Mark5
Mark6
```

平时读的 `log` 日志文件其实都是用这个命令输出的。

**输出重定向 `2>`**

标准错误输出

```shell
$ cat not_exist_file.csv > res.txt 2> errors.log
```

- 当 `cat` 一个文件时，会把文件内容打印到屏幕上，这个是标准输出；
- 当使用了 `> res.txt` 时，则不会打印到屏幕，会把标准输出写入文件 `res.txt` 文件中；
- `2> errors.log` 当发生错误时会写入 `errors.log` 文件中。

**输出重定向 `2>>`**

标准错误输出（追加到文件末尾）同 `>>` 相似。

**输出重定向 `2>&1`**

标准输出和标准错误输出都重定向都一个地方

```shell
$ cat not_exist_file.csv > res.txt 2>&1  # 覆盖输出
$ cat not_exist_file.csv >> res.txt 2>&1 # 追加输出
```

目前为止，接触的命令的输入都来自命令的参数，其实命令的输入还可以来自文件或者键盘的输入。

**输入重定向 `<`**

`<` 符号用于指定命令的输入。

```shell
cat < name.csv # 指定命令的输入为 name.csv
```

虽然它的运行结果与 `cat name.csv` 一样，但是它们的原理却完全不同。

- `cat name.csv` 表示 `cat` 命令接收的输入是 `notes.csv` 文件名，那么要先打开这个文件，然后打印出文件内容。
- `cat < name.csv` 表示 `cat` 命令接收的输入直接是 `notes.csv` 这个文件的内容， `cat` 命令只负责将其内容打印，打开文件并将文件内容传递给 `cat` 命令的工作则交给终端完成。

**输入重定向 `<<`**

将键盘的输入重定向为某个命令的输入。

```shell
sort -n << END    # 输入这个命令之后，按下回车，终端就进入键盘输入模式，其中END为结束命令（这个可以自定义）

wc -m << END      # 统计输入的单词
```

### 管道 `|`

把两个命令连起来使用，一个命令的输出作为另外一个命令的输入，英文是 `pipeline` ，可以想象一个个水管连接起来，管道算是重定向流的一种。

![图片](https://mmbiz.qpic.cn/mmbiz/9aPYe0E1fb0FicFcvBKLaKrNcIBiaVW1lRK1J5vBvY1mvOyXIBCBFdawPOUV3GgwX60P2vFMiaziaWv8hSPib6b1RaA/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

举几个实际用法案例：

```shell
cut -d , -f 1 name.csv | sort > sorted_name.txt 
# 第一步获取到的 name 列表，通过管道符再进行排序，最后输出到sorted_name.txt

du | sort -nr | head 
# du 表示列举目录大小信息
# sort 进行排序,-n 表示按数字排序，-r 表示倒序
# head 前10行文件

grep log -Ir /var/log | cut -d : -f 1 | sort | uniq
# grep log -Ir /var/log 表示在log文件夹下搜索 /var/log 文本，-r 表示递归，-I 用于排除二进制文件
# cut -d : -f 1 表示通过冒号进行剪切，获取剪切的第一部分
# sort 进行排序
# uniq 进行去重
```

### 流

流并非一个命令，在计算机科学中，流 `stream` 的含义是比较难理解的，记住一点即可：**流就是读一点数据, 处理一点点数据。其中数据一般就是二进制格式。** 上面提及的重定向或管道，就是把数据当做流去运转的。

到此我们就接触了，流、重定向、管道等 `Linux` 高级概念及指令。其实你会发现关于流和管道在其它语言中也有广泛的应用。 `Angular` 中的模板语法中可以使用管道。 `Node.js` 中也有 `stream` 流的概念。



## 进程管理

### 查看进程

在 `Windows` 中通过 `Ctrl + Alt + Delete` 快捷键查看软件进程。

#### w

帮助我们快速了解系统中目前有哪些用户登录着，以及他们在干什么。

```shell
[root@lion ~]# w
 06:31:53 up 25 days,  9:53,  1 user,  load average: 0.00, 0.01, 0.05
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
root     pts/0    118.31.243.53    05:56    1.00s  0.02s  0.00s w
 
06:31:53：表示当前时间
up 25 days, 9:53：表示系统已经正常运行了“25天9小时53分钟”
1 user：表示一个用户
load average: 0.00, 0.01, 0.05：表示系统的负载，3个值分别表示“1分钟的平均负载”，“5分钟的平均负载”，“15分钟的平均负载”

 USER：表示登录的用于
 TTY：登录的终端名称为pts/0
 FROM：连接到服务器的ip地址
 LOGIN@：登录时间
 IDLE：用户有多久没有活跃了
 JCPU：该终端所有相关的进程使用的 CPU 时间，每当进程结束就停止计时，开始新的进程则会重新计时
 PCPU：表示 CPU 执行当前程序所消耗的时间，当前进程就是在 WHAT 列里显示的程序
 WHAT：表示当下用户正运行的程序是什么，这里我运行的是 w
```

#### ps

用于显示当前系统中的进程， `ps` 命令显示的进程列表不会随时间而更新，是静态的，是运行 `ps` 命令那个时刻的状态或者说是一个进程快照。

**基础语法**：

```shell
[root@lion ~]# ps
  PID TTY          TIME CMD
 1793 pts/0    00:00:00 bash
 4756 pts/0    00:00:00 ps
 
 PID：进程号，每个进程都有唯一的进程号
 TTY：进程运行所在的终端
 TIME：进程运行时间
 CMD：产生这个进程的程序名，如果在进程列表中看到有好几行都是同样的程序名，那么就是同样的程序产生了不止一个进程
```

**常用参数**：

- `-w` 显示加宽可以显示较多的资讯
- `-A` 列出所有的进程
- `-ef` 列出所有进程（展示的列比`-A`多）;
- `-efH` 以乔木状列举出所有进程;
- `-u` 列出此用户运行的进程;
- `-au` 显示较详细的资讯（包括此用户和root用户）
- `-aux`  显示所有包含其他使用者的进程（比`-ef`多展示CPU和内存使用情况）
- `-aux --sort -pcpu` 按 `CPU` 使用降序排列， `-aux --sort -pmem` 表示按内存使用降序排列;
- `-axjf` 以树形结构显示进程， `ps -axjf` 它和 `pstree` 效果类似。

#### top

获取进程的动态列表：

```shell
top - 07:20:07 up 25 days, 10:41,  1 user,  load average: 0.30, 0.10, 0.07
Tasks:  67 total,   1 running,  66 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.7 us,  0.3 sy,  0.0 ni, 99.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  1882072 total,   552148 free,   101048 used,  1228876 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  1594080 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND                                                                                                       
  956 root      10 -10  133964  15848  10240 S  0.7  0.8 263:13.01 AliYunDun                                                                                                     
    1 root      20   0   51644   3664   2400 S  0.0  0.2   3:23.63 systemd                                                                                                       
    2 root      20   0       0      0      0 S  0.0  0.0   0:00.05 kthreadd                                                                                                      
    4 root       0 -20       0      0      0 S  0.0  0.0   0:00.00 kworker/0:0H    
```

- `top - 07:20:07 up 25 days, 10:41, 1 user, load average: 0.30, 0.10, 0.07` 相当 `w` 命令的第一行的信息。
- 展示的这些进程是按照使用处理器 `%CPU` 的使用率来排序的。

#### kill

结束一个进程， `kill + PID` 。

```shell
kill 956         # 结束进程号为956的进程
kill 956 957     # 结束多个进程
kill -9 7291     # 强制结束进程
```

### 管理进程

#### 进程状态

主要是切换进程的状态。我们先了解下 `Linux` 下进程的五种状态：

1. 状态码 `R` ：表示正在运行的状态；
2. 状态码 `S` ：表示中断（休眠中，受阻，当某个条件形成后或接受到信号时，则脱离该状态）；
3. 状态码 `D` ：表示不可中断（进程不响应系统异步信号，即使用kill命令也不能使其中断）；
4. 状态码 `Z` ：表示僵死（进程已终止，但进程描述符依然存在，直到父进程调用 `wait4()` 系统函数后将进程释放）；
5. 状态码 `T` ：表示停止（进程收到 `SIGSTOP` 、 `SIGSTP` 、 `SIGTIN` 、 `SIGTOU` 等停止信号后停止运行）。

#### 前台进程 & 后台进程

默认情况下，用户创建的进程都是前台进程，前台进程从键盘读取数据，并把处理结果输出到显示器。例如运行 `top` 命令，这就是一个一直运行的前台进程。

后台进程的优点是不必等待程序运行结束，就可以输入其它命令。在需要执行的命令后面添加 `&` 符号，就表示启动一个后台进程。

**`&`**

启动后台进程，它的缺点是后台进程与终端相关联，**一旦关闭终端，进程就自动结束了**。

```shell
$ cp name.csv name-copy.csv &
```

**`nohup`**

使进程不受挂断（关闭终端等动作）的影响。

```shell
$ nohup cp name.csv name-copy.csv
```

`nohup` 命令也可以和 `&` 结合使用。

```shell
$ nohup cp name.csv name-copy.csv &
```

**`bg`**

使一个“后台暂停运行”的进程，状态改为“后台运行”。

```shell
$ bg %1      # 不加任何参数的情况下，bg命令会默认作用于最近的一个后台进程，如果添加参数则会作用于指定标号的进程
```

实际案例1：

```shell
1. 执行 grep -r "log" / > grep_log 2>&1 命令启动一个前台进程，并且忘记添加 & 符号
2. ctrl + z 使进程状态转为后台暂停
3. 执行 bg 将命令转为后台运行
```

实际案例2：

```shell
前端开发时我们经常会执行 yarn start 启动项目
此时我们执行 ctrl + z 先使其暂停
然后执行 bg 使其转为后台运行
这样当前终端就空闲出来可以干其它事情了，如果想要唤醒它就使用 fg 命令即可（后面会讲）
```

#### jobs

显示当前终端后台进程状态。

```shell
[root@lion ~]# jobs
[1]+  Stopped                 top
[2]-  Running                 grep --color=auto -r "log" / > grep_log 2>&1 &
```

#### fg

`fg` 使进程转为前台运行，用法和 `bg` 命令类似。

我们用一张图来表示前后台进程切换：

![进程状态切换](https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/%E8%BF%9B%E7%A8%8B%E7%8A%B6%E6%80%81%E5%88%87%E6%8D%A2.jpg)

我们可以使程序在后台运行，成为后台进程，这样在当前终端中我们就可以做其他事情了，而不必等待此进程运行结束。

### 守护进程

一个运行起来的程序被称为进程。在 `Linux` 中有些进程是特殊的，它不与任何进程关联，不论用户的身份如何，都在后台运行，这些进程的父进程是 `PID` 为1的进程， `PID` 为1的进程只在系统关闭时才会被销毁。它们会在后台一直运行等待分配工作。我们将这类进程称之为守护进程 `daemon` 。

守护进程的名字通常会在最后有一个 `d` ，表示 `daemon` 守护的意思，例如 `systemd` 、`httpd` 。

#### systemd

`systemd` 是一个 `Linux` 系统基础组件的集合，提供了一个系统和服务管理器，运行为 `PID 1` 并负责启动其它程序。

```shell
[root@lion ~]# ps -aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.2  51648  3852 ?        Ss   Feb01   1:50 /usr/lib/systemd/systemd --switched-root --system --deserialize 22
```

通过命令也可以看到 `PID` 为1的进程就是 `systemd` 的系统进程。

`systemd` 常用命令（它是一组命令的集合）：

```shell
systemctl start nginx # 启动服务
systemctl stop nginx # 停止服务
systemctl restart nginx # 重启服务
systemctl status nginx # 查看服务状态
systemctl reload nginx # 重载配置文件(不停止服务的情况)
systemctl enable nginx # 开机自动启动服务
systemctl disable nginx # 开机不自动启动服务
systemctl is-enabled nginx # 查看服务是否开机自动启动
systemctl list-unit-files --type=service # 查看各个级别下服务的启动和禁用情况
```



## 文件压缩解压

- 打包：是将多个文件变成一个总的文件，它的学名叫存档、归档。
- 压缩：是将一个大文件（通常指归档）压缩变成一个小文件。

常常使用 `tar` 将多个文件归档为一个总的文件，称为 `archive` ， 然后用 `gzip` 或 `bzip2` 命令将 `archive` 压缩为更小的文件。

![文件打包压缩](https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/%E6%96%87%E4%BB%B6%E6%89%93%E5%8C%85%E5%8E%8B%E7%BC%A9.jpg)

### tar

创建一个 `tar` 归档。

**基础用法**：

```shell
$ tar -cvf sort.tar sort/                   # 将sort文件夹归档为sort.tar
$ tar -cvf archive.tar file1 file2 file3    # 将 file1 file2 file3 归档为archive.tar
```

**常用参数**：

- `-cvf` 表示 `create`（创建）+ `verbose`（细节）+ `file`（文件），创建归档文件并显示操作细节；
- `-tf` 显示归档里的内容，并不解开归档；
- `-rvf` 追加文件到归档， `tar -rvf archive.tar file.txt` ；
- `-xvf` 解开归档， `tar -xvf archive.tar` 。

### gzip / gunzip

“压缩/解压”归档，默认用 `gzip` 命令，压缩后的文件后缀名为 `.tar.gz` 。

```shell
gzip archive.tar          # 压缩
gunzip archive.tar.gz     # 解压
```

### tar 归档+压缩

可以用 `tar` 命令同时完成归档和压缩的操作，就是给 `tar` 命令多加一个选项参数，使之完成归档操作后，还是调用 `gzip` 或 `bzip2` 命令来完成压缩操作。

```shell
tar -zcvf archive.tar.gz archive/        # 将archive文件夹归档并压缩
tar -zxvf archive.tar.gz                 # 将archive.tar.gz归档压缩文件解压
```

### zcat、zless、zmore

之前讲过使用 `cat less more` 可以查看文件内容，但是压缩文件的内容是不能使用这些命令进行查看的，而要使用 `zcat、zless、zmore` 进行查看。

```shell
zcat archive.tar.gz
```

### zip/unzip

“压缩/解压” `zip` 文件（ `zip` 压缩文件一般来自 `windows` 操作系统）。

**命令安装**：

```shell
# Red Hat 一族中的安装方式
yum install zip 
yum install unzip 
```

**基础用法**：

```shell
unzip archive.zip # 解压 .zip 文件
unzip -l archive.zip # 不解开 .zip 文件，只看其中内容

zip -r sort.zip sort/ # 将sort文件夹压缩为 sort.zip，其中-r表示递归
```



## 编译安装软件

之前我们学会了使用 `yum` 命令进行软件安装，如果碰到 `yum` 仓库中没有的软件，就需要更高级的软件安装“源码编译安装”。

### wget

可以使我们直接从终端控制台下载文件，只需要给出文件的HTTP或FTP地址。

```shell
wget [参数][URL地址]

wget http://www.minjieren.com/wordpress-3.1-zh_CN.zip
```

`wget` 非常稳定，如果是由于网络原因下载失败， `wget` 会不断尝试，直到整个文件下载完毕。

**常用参数**：

- `-c` 继续中断的下载。

### 编译安装

简单来说，编译就是将程序的源代码转换成可执行文件的过程。大多数 `Linux` 的程序都是开放源码的，可以编译成适合我们电脑和操纵系统属性的可执行文件。

基本步骤如下：

1. 下载源代码

   我们来编译安装 `htop` 软件，首先在它的官网下载源码：https://bintray.com/htop/source/htop#files

   下载好的源码在本机电脑上使用如下命令同步到服务器上：

   ```shell
   $ scp 文件名 用户名@服务器ip:目标路径
   
   $ scp ~/Desktop/htop-3.0.0.tar.gz root@121.42.11.34:.
   ```

   也可以使用 `wget` 进行下载：

   ```shell
   $ wget+下载地址
   
   $ wget https://bintray.com/htop/source/download_file?file_path=htop-3.0.0.tar.gz
   ```

2. 解压压缩包

   ```shell
   $ tar -zxvf htop-3.0.0.tar.gz      # 解压
   
   $ cd htop-3.0.0                    # 进入目录
   ```

3. 配置

   执行 `./configure` ，它会分析你的电脑去确认编译所需的工具是否都已经安装了。

4. 编译

   执行 `make` 命令

5. 安装

   执行 `make install` 命令，安装完成后执行 `ls /usr/local/bin/` 查看是否有 `htop` 命令。如果有就可以执行 `htop` 命令查看系统进程了。



## 网络

### ifconfig

查看 `ip` 网络相关信息，如果命令不存在的话， 执行命令 `yum install net-tools` 安装。

```shell
[root@lion ~]# ifconfig

eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.31.24.78  netmask 255.255.240.0  broadcast 172.31.31.255
        ether 00:16:3e:04:9c:cd  txqueuelen 1000  (Ethernet)
        RX packets 1592318  bytes 183722250 (175.2 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1539361  bytes 154044090 (146.9 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

参数解析：

- `eth0` 对应有线连接（对应你的有线网卡），就是用网线来连接的上网。 `eth` 是 `Ethernet` 的缩写，表示“以太网”。有些电脑可能同时有好几条网线连着，例如服务器，那么除了 `eht0` ，你还会看到 `eth1` 、 `eth2` 等。
- `lo` 表示本地回环（ `Local Loopback` 的缩写，对应一个虚拟网卡）可以看到它的 `ip` 地址是 `127.0.0.1` 。每台电脑都应该有这个接口，因为它对应着“连向自己的链接”。这也是被称之为“本地回环”的原因。所有经由这个接口发送的东西都会回到你自己的电脑。看起来好像并没有什么用，但有时为了某些缘故，我们需要连接自己。例如用来测试一个网络程序，但又不想让局域网或外网的用户查看，只能在此台主机上运行和查看所有的网络接口。例如在我们启动一个前端工程时，在浏览器输入 `127.0.0.1:3000` 启动项目就能查看到自己的 `web` 网站，并且它只有你能看到。
- `wlan0` 表示无线局域网（上面案例并未展示）。

### host

`ip` 地址和主机名的互相转换。

**软件安装**：

```shell
yum install bind-utils
```

**基础用法**：

```shell
[root@lion ~]# host github.com
baidu.com has address 13.229.188.59
 
[root@lion ~]# host 13.229.188.59
59.188.229.13.in-addr.arpa domain name pointer ec2-13-229-188-59.ap-southeast-1.compute.amazonaws.com.
```

### ssh 连接远程服务器

通过非对称加密以及对称加密的方式（同 `HTTPS` 安全连接原理相似）连接到远端服务器。

```shell
ssh 用户@ip:port

1、ssh root@172.20.10.1:22 # 端口号可以省略不写，默认是22端口
2、输入连接密码后就可以操作远端服务器了
```

配置 **ssh**：

`config` 文件可以配置 `ssh` ，方便批量管理多个 `ssh` 连接。

配置文件分为以下几种：

- 全局 `ssh` 服务端的配置： `/etc/ssh/sshd_config` ；
- 全局 `ssh` 客户端的配置： `/etc/ssh/ssh_config`（很少修改）；
- 当前用户 `ssh` 客户端的配置： `~/.ssh/config` 。

【服务端 `config` 文件的常用配置参数】

| 服务端 config 参数     | 作用                                       |
| :--------------------- | :----------------------------------------- |
| Port                   | sshd 服务端口号（默认是22）                |
| PermitRootLogin        | 是否允许以 root 用户身份登录（默认是可以） |
| PasswordAuthentication | 是否允许密码验证登录（默认是可以）         |
| PubkeyAuthentication   | 是否允许公钥验证登录（默认是可以）         |
| PermitEmptyPasswords   | 是否允许空密码登录（不安全，默认不可以）   |

[注意] 修改完服务端配置文件需要重启服务 `systemctl restart sshd`

【客户端 `config` 文件的常用配置参数】

| 客户端 config 参数 | 作用                     |
| :----------------- | :----------------------- |
| Host               | 别名                     |
| HostName           | 远程主机名（或 IP 地址） |
| Port               | 连接到远程主机的端口     |
| User               | 用户名                   |

配置当前用户的 `config` ：

```shell
# 创建config
vim ~/.ssh/config

# 填写一下内容
Host lion # 别名
 HostName 172.x.x.x # ip 地址
  Port 22 # 端口
  User root # 用户
```

这样配置完成后，下次登录时，可以这样登录 `ssh lion` 会自动识别为 `root` 用户。

[注意] 这段配置不是在服务器上，而是你自己的机器上，它仅仅是设置了一个别名。

**免密登录**：

`ssh` 登录分两种，一种是基于口令（账号密码），另外一种是基于密钥的方式。

基于口令，就是每次登录输入账号和密码，显然这样做是比较麻烦的，今天主要学习如何基于密钥实现免密登录。

**基于密钥验证原理**：

客户机生成密钥对（公钥和私钥），把公钥上传到服务器，每次登录会与服务器的公钥进行比较，这种验证登录的方法更加安全，也被称为“公钥验证登录”。

**具体实现步骤**：

1、在客户机中生成密钥对（公钥和私钥） `ssh-keygen`（默认使用 RSA 非对称加密算法）

运行完 `ssh-keygen` 会在 `~/.ssh/` 目录下，生成两个文件：

- `id_rsa.pub` ：公钥
- `id_rsa` ：私钥

2、把客户机的公钥传送到服务

执行 `ssh-copy-id root@172.x.x.x`（`ssh-copy-id` 它会把客户机的公钥追加到服务器 `~/.ssh/authorized_keys` 的文件中）。

执行完成后，运行 `ssh root@172.x.x.x` 就可以实现免密登录服务器了。

配合上面设置好的别名，直接执行 `ssh lion` 就可以登录，是不是非常方便。



## 备份

### scp

它是 `Secure Copy` 的缩写，表示安全拷贝。 `scp` 可以使我们通过网络，把文件从一台电脑拷贝到另一台电脑。

`scp` 是基于 `ssh` 的原理来运作的， `ssh` 会在两台通过网络连接的电脑之间创建一条安全通信的管道， `scp` 就利用这条管道安全地拷贝文件。

```shell
$ scp source_file destination_file               # source_file 表示源文件，destination_file 表示目标文件
```

其中 `source_file` 和 `destination_file` 都可以这样表示： `user@ip:file_name` ， `user` 是登录名， `ip` 是域名或 `ip` 地址。 `file_name` 是文件路径。

```shell
$ scp file.txt root@192.168.1.5:/root             # 表示把我的电脑中当前文件夹下的 file.txt 文件拷贝到远程电脑
$ scp root@192.168.1.5:/root/file.txt file.txt    # 表示把远程电脑上的 file.txt 文件拷贝到本机
```

### rsync

`rsync` 命令主要用于远程同步文件。它可以同步两个目录，不管它们是否处于同一台电脑。它应该是最常用于“增量备份”的命令了。它就是智能版的 `scp` 命令。

**软件安装**：

```shell
$ yum install rsync
```

**基础用法**：

```shell
$ rsync -arv Images/ backups/ # 将Images 目录下的所有文件备份到 backups 目录下
$ rsync -arv Images/ root@192.x.x.x:backups/ # 同步到服务器的backups目录下
```

**常用参数**：

- `-a` 保留文件的所有信息，包括权限，修改日期等；
- `-r` 递归调用，表示子目录的所有文件也都包括；
- `-v` 冗余模式，输出详细操作信息。

默认地， `rsync` 在同步时并不会删除目标目录的文件，例如你在源目录中删除一个文件，但是用 `rsync` 同步时，它并不会删除同步目录中的相同文件。如果向删除也可以这么做： `rsync -arv --delete Images/ backups/` 。



## 系统

### halt

关闭系统，需要 `root` 身份。

```shell
$ halt
```

### reboot

重启系统，需要 `root` 身份。

```shell
$ reboot
```

### poweroff

直接运行即可关机，不需要 `root` 身份。
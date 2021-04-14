## Mac开发配置

### 一、 系统设置

操作系统中，首先需要做一件事就是**更新系统**，点击窗口左上角的 ** > 关于本机 > 软件更新** 。此外还需要到系统设置进行一些适当调整：

1. 触控板

   系统设置 > 触控板

   - 光标与点击
     - ✓ 轻拍来点按
     - ✓ 辅助点按
     - ✓ 查找
     - ✓ 三指拖移
   - 滚动缩放
     - ✓ 默认全选
   - 更多手势
     - ✓ 默认全选

2. Dock
   - 置于屏幕上的位置：左边
   - 设置 Dock 图标更小（大小随个人喜好）
   - ✓ 自动显示和隐藏 Dock

3. Finder
   - Finder > 显示
     - 显示标签页栏
     - 显示路径栏
     - 显示状态栏
     - 自定工具栏 > 去除所有按钮，仅剩搜索栏
- Finder > 偏好设置
     - 通用
       - 开启新 Finder 窗口时打开：HOME「用户名」目录
     - 边栏
       - 添加 HOME「用户名」目录 和 创建代码文件目录
       - 将 共享的(shared) 和 标记(tags) 目录去掉

4. 菜单栏
   - 去掉蓝牙等无需经常使用的图标
   - 将电池显示设置为百分比

5. Spotlight
   - 去掉字体和书签与历史记录等不需要的内容
   - 设置合适的快捷键

6. 互联网帐户
   - 添加 iCloud 用户，同步日历，联系人和 Find my mac 等等

### 二、 Xcode

安装 Xcode command line tools：

```
xcode-select --install
```

### 三、 Homebrew

在安装 Homebrew 之前，需要将 `Xcode Command Line Tools` 安装完成，这样就可以使用基于 `Xcode Command Line Tools` 编译的 Homebrew。

在 terminal 中复制以下命令，跟随指引，将完成 Hombrew 安装。

```
$ ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

紧接着，需要让通过 Homebrew 安装的程序的启动链接 (在 `/usr/local/bin`中）可以直接运行，无需将完整路径写出。通过以下命令将 `/usr/local/bin` 添加至 `$PATH` 环境变量中:

```
$ echo 'export PATH="/usr/local/bin:$PATH"' >> ~/.bash_profile
```

**Cmd+T** 打开一个新的 terminal 标签页，运行以下命令，确保 brew 运行正常。

```
$ brew doctor
```

安装完成后，Homebrew 会将本地 `/usr/local` 初始化为 git 的工作树，并将目录所有者变更为当前所操作的用户，将来 `brew` 的相关操作不需要 sudo 。

### 四、 ITerm2

[ITerm2 - 让Mac命令行更加丰富高效](https://www.jianshu.com/p/405956cdaca6)

### 五、梯子



### 六、 常用系统软件

Mac常用软件下载地址：[Xclient](https://xclient.info/)

- 开发者工具
  - [Google Chrome](https://www.google.com/intl/en/chrome/browser/)
  - [Idea](https://www.jetbrains.com/zh-cn/idea/)
  - [Sublime Text](https://www.sublimetext.com/3)
  - [Navicat Premium](https://xclient.info/s/navicat-premium.html)
- 生产力工具
  - [CleanMyMac](https://xclient.info/s/cleanmymac.html) / [App Uninstaller](https://xclient.info/s/app-uninstaller.html)
  - [Unarchiver](http://wakaba.c3.cx/s/apps/unarchiver.html)
  - [Tuxera NTFS](https://xclient.info/s/tuxera-ntfs.html)
- 办公工具
  - [Microsoft Office](https://www.microsoft.com/zh-cn/microsoft-365/mac/microsoft-365-for-mac?rtc=1)
  - [XMind](https://www.xmind.net/)
  - [Typora](https://typora.io/#download)
- 其他
  - [Parallels Desktop](https://www.parallels.com/hk/)
  - [VLC Player](https://www.videolan.org/vlc/)


# Tmux学习


# Tmux学习

## 一.什么是Tmux

### 1.1传统命令行

命令行的典型使用方式是，打开一个终端窗口（terminal window，以下简称"窗口"），在里面输入命令。**用户与计算机的这种临时的交互，称为一次"会话"（session）** 。

会话的一个重要特点是，窗口与其中启动的进程是连在一起的。打开窗口，会话开始；关闭窗口，会话结束，会话内部的进程也会随之终止，不管有没有运行完。

一个典型的例子就是，SSH 登录远程计算机，打开一个远程窗口执行命令。这时，网络突然断线，再次登录的时候，是找不回上一次执行的命令的。因为上一次 SSH 会话已经终止了，里面的进程也随之消失了。

为了解决这个问题，会话与窗口可以"解绑"：窗口关闭时，会话并不终止，而是继续运行，等到以后需要的时候，再让会话"绑定"其他窗口。

### 1.2Tmux作用

**将会话与窗口解绑的工具**

>功能

- 允许在单个窗口中，同时访问多个会话。
- 可以让新窗口接入已经存在的会话
- 允许每个会话有多个连接窗口，可以多人实时共享会话
- 支持窗口任意的垂直和水平拆分

---

## 二.基本用法

### 2.1安装

> centos系统下的安装

`sudo yum install tmux`

### 2.2启动和退出

> 启动

`tmux`

启动Tmux窗口,底部有一个状态栏。状态栏左侧是窗口信息(编号和名称)，右侧是系统信息

![image-20211002142538408](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211002142538408.png)

> 退出

`exit`或者Ctrl + D

### 2.3前缀键

Tmux的快捷键需要前缀键进行唤醒，默认前缀键是 Ctrl + B。先按下前缀键，快捷键才会生效

---

## 三.会话管理

### 3.1新建会话

默认启动的Tmux窗口编号从0开始，我们可以自己给会话起名字

`tmux new -s <session-name>`

### 3.2分离会话

> 分离会话

在Tmux窗口下，按住Ctrl + B D或者输入`tmux detach`，就会将当前会话和窗口分离，上面命令执行后，就会退出当前的Tmux窗口，但是后台仍在运行里面的会话和进程

> 查看所有会话

`tmux ls`

### 3.3接入会话

用于重新接入某个已经存在的会话

`tmux attach -t  <session-name>`

### 3.4杀死会话

`tmux kill-session -t <session-name>`

### 3.5切换会话

`tmux switch -t <session-name>`

### 3.6重命名会话

`tmux rename-session -t <session-name> <new-name>`

### 3.7会话快捷键

- Ctrl + B  D：分离当前会话
- Ctrl + B  S：列出所有会话
- Ctrl + B  $：重命名当前会话

---

## 四.最简操作流程

1. 新建会话`tmux new -s <session-name>`
2. 在Tmux窗口运行所需的程序
3. 按下快捷键`Ctrl +B  D`将会话分离
4. 下次使用时，重新连接到会话`tmux attach-session -t <session-name>`

---

## 五.网格操作

Tmux可以将窗口分成多个窗格，每个窗格运行不同的命令

### 5.1划分窗格

```bash
tmux split-window #划分上下两个窗格
tmux split-window -h #划分左右两个窗格
```

![image-20211002151708877](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211002151708877.png)

### 5.2移动光标

使用`tmux select-pane`命令用来移动光标位置

```bash
tmux select-pane -U #光标切换到上方窗格
tmux select-pane -D #光标切换到下方窗格
tmux select-pane -L #光标切换到左边窗格
tmux select-pane -R #光标切换到右边窗格
```

### 5.3交换窗格位置

`tmux swap-pane` 命令来交换窗格位置

```bash
tmux swap-pane -U #当前窗格上移
tmux swap-pane -D #当前窗格下移
```

### 5.4窗格快捷键

- `Ctrl+b %`：划分左右两个窗格。
- `Ctrl+b "`：划分上下两个窗格。
- `Ctrl+b <arrow key>`：光标切换到其他窗格。`<arrow key>`是指向要切换到的窗格的方向键，比如切换到下方窗格，就按方向键`↓`。
- `Ctrl+b ;`：光标切换到上一个窗格。
- `Ctrl+b o`：光标切换到下一个窗格。
- `Ctrl+b {`：当前窗格与上一个窗格交换位置。
- `Ctrl+b }`：当前窗格与下一个窗格交换位置。
- `Ctrl+b Ctrl+o`：所有窗格向前移动一个位置，第一个窗格变成最后一个窗格。
- `Ctrl+b Alt+o`：所有窗格向后移动一个位置，最后一个窗格变成第一个窗格。
- `Ctrl+b x`：关闭当前窗格。
- `Ctrl+b !`：将当前窗格拆分为一个独立窗口。
- `Ctrl+b z`：当前窗格全屏显示，再使用一次会变回原来大小。
- `Ctrl+b Ctrl+<arrow key>`：按箭头方向调整窗格大小。
- `Ctrl+b q`：显示窗格编号

---

## 六.窗口管理

Tmux允许新建多个窗口

### 6.1新建窗口

`tmux new-window`命令用来创建新窗口

```bash
tmux new-window #新建一个窗口
tmux new-wndow -n <window-name> #新建一个指定名称的窗口
```

### 6.2切换窗口

`tmux select-window`命令用来切换窗口

```bash
tmux select-window -t <window-number> #切换到指定编号的窗口
tmux select-window -t <widnow-name> #切换到指定编号的窗口
```

### 6.3重命名窗口

`tmux rename-window`命令用于为当前窗口起名(或重命名)

```bash
tmux rename-window <new-name>
```

### 6.4窗口快捷键

- `Ctrl+b c`：创建一个新窗口，状态栏会显示多个窗口的信息。
- `Ctrl+b p`：切换到上一个窗口（按照状态栏上的顺序）。
- `Ctrl+b n`：切换到下一个窗口。
- `Ctrl+b <number>`：切换到指定编号的窗口，其中的`<number>`是状态栏上的窗口编号。
- `Ctrl+b w`：从列表中选择窗口。
- `Ctrl+b ,`：窗口重命名。

---

## 七.其他命令

```bash
# 列出所有快捷键，及其对应的 Tmux 命令
tmux list-keys

# 列出所有 Tmux 命令及其参数
tmux list-commands

# 列出当前所有 Tmux 会话的信息
tmux info

# 重新加载当前的 Tmux 配置
tmux source-file ~/.tmux.conf
```

---

## 八.参考连接

- https://www.ruanyifeng.com/blog/2019/10/tmux.html



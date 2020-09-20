## TMUX基本操作和配置整理

### 简介

TMUX指**terminal multiplexer**，即终端复用软件。tmux结构包含以下三个部分：

* **session**  —— 会话，可以用tmux创建多个会话。
* **window** —— 窗口， 一个会话中可以包含多个窗口。
* **pane** —— 窗格，用于分隔窗口，一个窗口中可以包含多个窗格。

### 安装

```shell
apt-get install tmux
```

 ### 基本操作

<font color = 'red'>**说明：**\<C-b> 指 Ctrl+b键; \<C-b> d指先按Ctrl+b键, 再按d键，不是指同时按下Ctrl+b和d</font>

#### session操作

```shell
tmux new -s <session>					#创建新会话
tmux new -s <session> -d				#后台创建会话
tmux ls									#列出所有会话
tmux a -t <session>						#回到某个会话,a指attach
tmux rename -t <old_name> <new_name>	#将指定会话改名
tmux kill-session -t <session>			#关闭某个会话
tmux kill-server						#重启所有tmux进程
<C-b> d									#暂时离开tmux,回到终端,d指detach
<C-b> s									#选择会话列表
<C-b> $									#重命名当前会话
```

#### window操作

```shell
<C-b> w									#列出所有窗口
<C-b> <C-o>								#切换窗口顺序
<C-b> 0-9								#选择几号窗口
<C-b> p									#切换上一个窗口, p指previous
<C-b> n									#切换下一个窗口, n指next
<C-d>									#退出tmux窗口, 相当于敲exit
<C-b> &									#退出当前窗口, 关闭所有窗格
<C-b> ,									#给窗口改名
```

#### pane操作

```shell
<C-b> %									#纵向分隔窗口
<C-b> " 								#横向分隔窗口
<C-b> <Up/Down/Left/Right>				#方向键切换窗格，可通过配置改成HJKL
<C-b> z									#最大化当前窗格
<C-b> x									#关闭当前使用中的窗格
<C-b> q									#显示序号,在序号消失前按对应序号可切换到对应窗格
<C-b> o									#顺时针切换窗口
<C-b> <C-o>								#逆时针切换窗口
```

#### 上下滚动，查看历史 

先按<C-b>, 再按[键，进入复制模式后，用PgUp, PgDn查看历史，再按q退出。

### 配置

添加~/.tmux.conf，修改内容如下：

```shell
# 定义快捷键<C-b> r, 快速加载tmux配置文件
bind r source-file ~/.tmux.conf \; display "tmux.conf reload!"

# 适应VIM操作，上下左右改为h,j,k,l
bind h select-pane -L
bind j select-pane -D
bind k select-pane -U
bind l select-pane -R

# 更改横分屏，竖分屏键位
bind | split-window -h
bind - split-window -v

# 设置窗口、窗格起始序号为1
set -g base-index 1
set -g pane-base-index 1
```

### 参考文档

TMUX常用快捷键和问题 —— <a href="https://www.cnblogs.com/piperck/p/4992159.html">https://www.cnblogs.com/piperck/p/4992159.html</a>


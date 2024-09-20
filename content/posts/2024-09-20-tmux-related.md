# tmux 配置使用

## 1、What's tmux

[tmux](http://tmux.github.io/) 是一个终端复用器: 可以激活多个终端或窗口, 在每个终端都可以单独访问，每一个终端都可以访问，运行和控制各自的程序.tmux类似于screen，可以关闭窗口将程序放在后台运行，需要的时候再重新连接。

## 2、How to install tmux

如果你创建的是workspace，那么已经安装好了tmux，如果你想全新安装tmux，只需要执行：

`sudo` `apt-get install` `-y tmux`

## 3、配置tmux

### **3.1、编辑 ~/.tmux.conf 文件，添加下面代码**

```python
set -sg escape-time 0
set-option -g history-limit 30000
# set-option -g default-shell /bin/zsh  # 使用 zsh 为默认 shell
set-window-option -g mode-keys vi # vi key
set-option -g status-keys vi
set -g default-terminal "tmux-256color"
  
# vim-like pane selection
bind l select-pane -R
bind j select-pane -D
bind k select-pane -U
bind h select-pane -L
  
bind -r c-h resize-pane -L 5
bind -r c-j resize-pane -D 1
bind -r c-k resize-pane -U 1
bind -r c-l resize-pane -R 5
  
# 在当前目录创建新窗口
unbind-key c
bind c new-window -c "#{pane_current_path}"
unbind-key '"'
unbind-key '%'
bind '"' split-window -c '#{pane_current_path}'
bind '%' split-window -h -c '#{pane_current_path}'
# end
  
set -g base-index 1 # start windows numbering at 1
setw -g pane-base-index 1 # make pane numbering consistent with windows
set-option -g update-environment "DBUS_SESSION_BUS_ADDRESS DISPLAY SSH_ASKPASS SSH_AUTH_SOCK SSH_AGENT_PID SSH_CONNECTION WINDOWID XAUTHORITY"
  
# 显示工作区标题
set -g pane-border-status top
set -g pane-border-format "#{pane_index} #T"
```

### **3.2、重新导入tmux配置环境**

`tmux source` `~/.tmux.conf`

## 4、如何让zsh标题为目录名？

编辑 ~/.zshrc 加入如下代码段：

```python
function set_tmux_title () {
    printf '\033]2;'"$1"'\033\\'
}
  
function auto_tmux_title() {
    emulate -L zsh
    printf '\033]2;'"${PWD:t}"'\033\\'
}
  
auto_tmux_title
chpwd_functions=(${chpwd_functions[@]} "auto_tmux_title")
```

## 5、Tmux cheet sheet

参考：[https://gist.github.com/ryerh/14b7c24dfd623ef8edc7](https://gist.github.com/ryerh/14b7c24dfd623ef8edc7)
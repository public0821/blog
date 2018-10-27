## 安装docker

参考官网

## 配置容器

1. 启动容器
```
$ docker run -dit -v /container/shared/:/shared --name dev -h dev debian
4ed302ce79dc950823f498833f24627f9474da2f89fbab0b5195159d685e283c
$ docker exec -it dev bash
root@dev:/#
```

2. 配置Debian源
```
# 先安装vim
root@dev:/# apt-get update
# vim-nox包含了对python, ruby的支持
root@dev:/# apt-get install vim-nox

# 配置软件源，这样下次安装东西的时候快一些
root@dev:/# vim /etc/apt/sources.list
# 使用这个源：http://mirrors.ustc.edu.cn/debian
root@dev:/# apt-get update
```

3. 安装一些必须的软件
```
root@dev:/# apt-get install git curl
```

## python

1. 创建一个python账户
```
root@dev:/# adduser python
root@dev:/# su - python
python@dev:~$ 
```

2. 安装一些必须的软件
```
root@dev:/# apt-get python3
```

3. 安装vundle
```
# 安装vundle
python@dev:~$ git clone https://github.com/VundleVim/Vundle.vim.git ~/.vim/bundle/Vundle.vim
```

4. 配置vimrc
```
python@dev:~$ cp python.vimrc .vimrc
# jedi的默认快捷键
let g:jedi#goto_command = "<leader>d"
let g:jedi#goto_assignments_command = "<leader>g"
let g:jedi#goto_definitions_command = ""
let g:jedi#documentation_command = "K"
let g:jedi#usages_command = "<leader>n"
let g:jedi#completions_command = "<C-Space>"
let g:jedi#rename_command = "<leader>r"
```

5. 安装插件
```
python@dev:~$ vim +PluginInstall +qall
```

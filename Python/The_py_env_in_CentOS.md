# CentOS下python开发环境部署

## Ⅰ、Python版本升级
### 背景：CentOS 6自带Python版本为2.6的，业内潜规则是不用2.7之前的Python,固我们有必要做一个升级
### 升级步骤：
- 查看原版本
``` 
[root@VM_98_88_centos ~]# python -V
Python 2.6.6
```     

- 下载Python 2.7安装包并编译安装
``` 
安装依赖包：
yum install -y gcc openssl-devel git readline-devel

下载源码包：
cd /usr/local/src
wget https://www.python.org/ftp/python/2.7.3/Python-2.7.3.tar.bz2
tar jxvf Python-2.7.3.tar.bz2

编译：
cd Python-2.7.3
./configure --prefix=/usr/local/python27

修改安装文件源码：(不改后面安装pip会报错)
vim /usr/local/src/Python-2.7.3/Modules/Setup   去掉两处注释后如下：
①zlib块 
# See http://www.gzip.org/zlib/
zlib zlibmodule.c -I$(prefix)/include -L$(exec_prefix)/lib -lz
②ssl块
# Socket module helper for socket(2)
_socket socketmodule.c

# Socket module helper for SSL support; you must comment out the other
# socket line above, and possibly edit the SSL variable:
SSL=/usr/local/ssl
_ssl _ssl.c \
        -DUSE_SSL -I$(SSL)/include -I$(SSL)/include/openssl \
        -L$(SSL)/lib -lssl -lcrypto

安装：
make && make install

验证：
[root@VM_98_88_centos Python-2.7.3]# ll /usr/local/python27
total 16
drwxr-xr-x 2 root root 4096 Oct 15 16:58 bin
drwxr-xr-x 3 root root 4096 Oct 15 16:58 include
drwxr-xr-x 4 root root 4096 Oct 15 16:58 lib
drwxr-xr-x 3 root root 4096 Oct 15 16:58 share
``` 

- 覆盖原Python链接
``` 
mv /usr/bin/python /usr/bin/python_bak
ln -s  /usr/local/python27/bin/python /usr/bin/

再瞅瞅：
[root@VM_98_88_centos ~]# python -V
Python 2.7.3     
``` 

- 修复yum
目前为止，Python升级已经完成了，但由于yum现在不支持Python 2.7，所以此时使用yum会报错
``` 
解决方法：将yum中的python环境变量指向原来的python 2.6
sed -i '1s/python/python_bak/g' /usr/bin/yum
```
## Ⅱ、关于pip
### 背景：为了便于管理第三方库，python提供了pip这个包管理工具，Python 2.7.9+和Python3.4+已经内置了pip，之前的版本需要手动部署
### 部署步骤：
- 下载安装
```
cd /usr/local/src
wget https://bootstrap.pypa.io/get-pip.py
python get-pip.py   (这里报错zlib包和ssl包问题原因是前面没修改安装包源码)
ln -s /usr/local/python27/bin/pip /usr/bin/pip
```

- 给pip提速
pypi.python.org网络比较不稳定，我们换用国内的pypi镜像
```
mkdir ~/.pip
cat >> ~/.pip/pip.conf << EOF
[global]
index-url = http://pypi.douban.com/simple/

[list]
format = columns
EOF
```
- pip日常用法
```
pip命令补全配置：
pip completion --bash >> ~/.bashrc
source !$

查已安装包列表
pip list

查找包：
pip search xxx

安装：
pip install xxx = 版本号

卸载:
pip uninstall xxx

查已安装包的信息：
pip show xxx

检查包是否缺少依赖：
pip check xxx

将已安装包列表导出至requirements文件：
pip freeze > requirements.txt

根据requirements文件安装包：
pip install -r requirements.txt

装个ipython玩玩
pip install ipython
ln -s  /usr/local/python27/bin/ipython /usr/bin/
```

## Ⅲ、vim中写python
### 背景：这里只做一个最基本的入门配置，包括插件管理器、智能提示、代码补全、语法检测、一键执行等
### 部署步骤：
- 安装vim-plug插件管理器
```
curl -fLo ~/.vim/autoload/plug.vim --create-dirs \
    https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim

cat >> ~/.vimrc << EOF
call plug#begin('~/.vim/plugged')
"Plug 'xxx'
call plug#end()
EOF
```

- 智能提示
```
Step1：
pip install jedi

Step2：
vim ~/.vimrc 在Plug插件管理部分添加两个插件配置如下：
Plug 'davidhalter/jedi-vim'
Plug 'ervandew/supertab'
"下面两行配置写在插件管理块外面，即call plug#end()之后
let g:SuperTabDefaultCompletionType = "context"
let g:jedi#popup_on_dot = 0

Step3：
:w保存  :PlugInstall 自动安装
```

- 代码补全
```
Step1：
vim ~/.vimrc 在Plug插件管理部分添加如下插件
Plug 'msanders/snipmate.vim'
:w保存  :PlugInstall 自动安装

Step2:
修改键位冲突，将此快捷键调为ctrl+n
vim /root/.vim/plugged/snipmate.vim/after/plugin/snipMate.vim
ino <silent> <tab> <c-r>=TriggerSnippet()<cr>
snor <silent> <tab> <esc>i<right><c-r>=TriggerSnippet()<cr>
改为：
ino <silent> <C-n> <c-r>=TriggerSnippet()<cr>
snor <silent> <C-n> <esc>i<right><c-r>=TriggerSnippet()<cr>
：wq

Step3：
定制补全举例：
vim /root/.vim/plugged/snipmate.vim/snippets/python.snippets
找到：
snippet #!
        #!/usr/bin/env python
添加：
        # -*- coding: UTF-8 -*-
：wq
```

- 语法检测
```
Step1：
vim ~/.vimrc 在Plug插件管理部分添加如下插件:
Plug 'scrooloose/syntastic'
"加入如下github推荐配置
set statusline+=%#warningmsg#
set statusline+=%{SyntasticStatuslineFlag()}
set statusline+=%*

let g:syntastic_always_populate_loc_list = 1
let g:syntastic_auto_loc_list = 1
let g:syntastic_check_on_open = 1
let g:syntastic_check_on_wq = 0

Step2：
:w保存  :PlugInstall 自动安装
```

- ~/.vimrc中配置一键执行
```
cat >> ~/.vimrc << EOF
map <F5> :call CompileRunGcc()<CR>
func! CompileRunGcc()
    exec "w"
    if &filetype == 'c'
        exec "!g++ % -o %<"
        exec "!time ./%<"
    elseif &filetype == 'cpp'
        exec "!g++ % -o %<"
        exec "!time ./%<"
    elseif &filetype == 'java'
        exec "!javac %"
        exec "!time java %<"
    elseif &filetype == 'sh'
        :!time bash %
    elseif &filetype == 'python'
        exec "!time python %"
    elseif &filetype == 'html'
        exec "!firefox % &"
    elseif &filetype == 'go'
"        exec "!go build %<"
        exec "!time go run %"
    elseif &filetype == 'mkd'
        exec "!~/.vim/markdown.pl % > %.html &"
        exec "!firefox %.html &"
    endif
endfunc
EOF
```

- 其他插件与配制
```
vim ~/.vimrc 在Plug插件管理部分添加如下插件:
"简单主题
Plug 'vim-airline/vim-airline'
Plug 'vim-airline/vim-airline-themes'
"自动缩进
Plug 'vim-scripts/indentpython.vim'
"PEP8代码风格检查
Plug 'nvie/vim-flake8'

"简单主题设置
let g:airline#extensions#tabline#enabled = 1
let g:airline_theme='simple'

"tab自动缩进4
set ts=4
set expandtab
set autoindent

"支持UTF-8编码
set encoding=utf-8

"显示行号
set nu

"python代码高亮
let python_highlight_all=1
syntax on

:w保存  :PlugInstall 自动安装插件

tips：调整回车后的tab长度为4(直接tab设为4不管用，这种还是8，蛋疼)
vim /root/.vim/plugged/indentpython.vim/indent/python.vim
加上下面这句：
setlocal et sta sw=4 sts=4
:wq
```

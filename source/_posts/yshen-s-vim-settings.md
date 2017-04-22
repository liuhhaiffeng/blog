---
title: yshen's vim settings
comments: true
date: 2016-12-14 01:59:33
tags: [工具, vim]
categories: 技术分享
---



> 写代码的时候手需要时刻保持在键盘上,随机定位代码、随机删除代码、移动代码、插入代码的操作大大多于阅读、翻页操作，中间卡顿一下效率就大大降低了；vim通过其强大的快捷键，可以完全摆脱鼠标。真是编程利器。通过插件，可以实现强大的IDE的功能。可以说每一个geek都有自己的vim配置，在这里我把我的配置共享出来。

![image_1b3rg746h1dpr29dsdu74o1u4qm.png-396.6kB][1]

安装的插件有

1. 插件管理
   Vundle
2. 索引提示
   ctag             生成工程的符号文件
   cscope           生成工程符号文件引用关系
   Tagbar           右边栏显示文件中的符号
3. 自动完成
   echofunc.vim     显示函数定义
   OmniCppComplete  提示结构体成员
   SuperTab         TAB键自动完成
4. 文件浏览
   The-NERD-tree
   ctrlp.vim
5. 其他
   molokai          配色主题
   c-standard-functions-highlight  高亮C语言函数


使用方法：
1. 安装Vundle
2. 复制如下脚本到vimrc
3. 进入vim之后运行:PluginUpdate

配置文件开源在git上：https://github.com/shenyuflying/vim

<!--more-->
以下内容可能过期

```
"-----------------------------------------------------------
"                yshen's vim settings
"----------------------------------------------------------
"
" PLUGINS
" ---------
" 1.插件管理
"   Vundle
" 2.索引提示
"   ctag             生成工程的符号文件
"   cscope           生成工程符号文件引用关系
"   Tagbar           右边栏显示文件中的符号
" 3.自动完成
"   echofunc.vim     显示函数定义
"   OmniCppComplete  提示结构体成员
"   SuperTab         TAB键自动完成
" 4.文件浏览
"   The-NERD-tree
"   ctrlp.vim
" 5.其他
"   molokai          配色主题
"   c-standard-functions-highlight  高亮C语言函数
"
"
" HOWTO
" ----------
" 1.快捷键映射
"   <F1>
"   <F2> 显示目录窗口
"   <F3> 显示符号窗口
"   <F4> 生成工程的符号文件
"
"   <F5> 高亮文本中的符号
"   <F6>
"   <F7>
"   <F8>
"
"   <F9>
"   <F10>
"   <F11>
"   <F12>
"
" INSTALL
" ---------
" 1. Install Vundle first
" 2. Install plugin by Vundle using :PluginUpdate
" 3. Copy this setting in your ~/.vimrc
"
"-----------------------------------------------------------
"                Vundle settings
"----------------------------------------------------------
set nocompatible              " be iMproved, required
filetype off                  " required

" set the runtime path to include Vundle and initialize
set rtp+=~/.vim/bundle/Vundle.vim
call vundle#begin()
" alternatively, pass a path where Vundle should install plugins
"call vundle#begin('~/some/path/here')

" let Vundle manage Vundle, required
Plugin 'VundleVim/Vundle.vim'
"Plugin 'Valloric/YouCompleteMe'

" The following are examples of different formats supported.
" Keep Plugin commands between vundle#begin/end.
" plugin on GitHub repo
Plugin 'tpope/vim-fugitive'

Plugin 'vim-airline/vim-airline'

" plugin from http://vim-scripts.org/vim/scripts.html
Plugin 'L9'
" Git plugin not hosted on GitHub
"Plugin 'git://git.wincent.com/command-t.git'
" git repos on your local machine (i.e. when working on your own plugin)
"Plugin 'file:///home/gmarik/path/to/plugin'
" The sparkup vim script is in a subdirectory of this repo called vim.
" Pass the path to set the runtimepath properly.
"Plugin 'rstacruz/sparkup', {'rtp': 'vim/'}
" Install L9 and avoid a Naming conflict if you've already installed a
" different version somewhere else.
"Plugin 'ascenator/L9', {'name': 'newL9'}
"Plugin 'taglist.vim'
Plugin 'Tagbar'
Plugin 'The-NERD-tree'
Plugin 'c-standard-functions-highlight'
Plugin 'molokai'
Plugin 'ctrlp.vim'
Plugin 'echofunc.vim'
Plugin 'OmniCppComplete'
"Plugin 'Syntastic'
Plugin 'SuperTab'

" All of your Plugins must be added before the following line
call vundle#end()            " required
filetype plugin indent on    " required
" To ignore plugin indent changes, instead use:
"filetype plugin on
"
" Brief help
" :PluginList       - lists configured plugins
" :PluginInstall    - installs plugins; append `!` to update or just :PluginUpdate
" :PluginSearch foo - searches for foo; append `!` to refresh local cache
" :PluginClean      - confirms removal of unused plugins; append `!` to auto-approve removal
"
" see :h vundle for more details or wiki for FAQ
" Put your non-Plugin stuff after this line
"
"-----------------------------------------------------------
"                generic vim settings
"----------------------------------------------------------

" 显示行号
set nu
" tab键为4个空格
set ts=4

"忽略大小写检索
"set ignorecase
"输入检索时动态变化
set incsearch
"检索高亮
set hlsearch
" 突出显示当前行
"set cursorline 
" 打开状态栏标尺
set ruler 
" 语法高亮
syntax on
" 在INSERT模式下移动
inoremap <C-h> <Left>
inoremap <C-j> <Down>
inoremap <C-k> <Up>
inoremap <C-l> <Right>
" 设置匹配模式，显示匹配的括号
set showmatch 
" 使用鼠标
"set mouse=a
"自动缩进
"set shiftwidth=4
"set softtabstop=4
"set tabstop=4
"set cindent
"set autoindent
"set noautoindent
"set cinoptions={0,1s,t0,n-2,p2s,(03s,=.5s,>1s,=1s,:1s
set smartindent


"-----------------------------------------------------------
"                build
"----------------------------------------------------------
"map <F5> :!make<CR>
"-----------------------------------------------------------
"                ctags
"----------------------------------------------------------
set tags=./
set tags=tags;
"生成tag cscope tags.vim等符号文件
map <F4> :!ctags -R --c++-kinds=+p --fields=+iaS --extra=+q . && cscope -Rbq  &&  awk '{print "syntax keyword tag "$1}' tags > tags.vim <CR>
"映射F4在插入模式下也可以用
imap <F4> <ESC>:!ctags -R --c++-kinds=+p --fields=+iaS --extra=+q . && cscope -Rbq  &&  awk '{print "syntax keyword tag "$1}' tags > tags.vim<CR>
map <F5> :so tags.vim<CR>
imap <F5> <ESC> :so tags.vim<CR>


"-----------------------------------------------------------
"                tagbar
"----------------------------------------------------------
"type help: tagbar for more info
let g:tagbar_ctags_bin='ctags'          "ctags程序的路径
let g:tagbar_width=30                   "窗口宽度的设置
"let g:tagbar_left = 1
let g:tagbar_right = 1
map <F3> :Tagbar<CR>
"autocmd BufReadPost *.cpp,*.c,*.h,*.hpp,*.cc,*.cxx call tagbar#autoopen() "如果是c语言的程序的话，tagbar自动开启
"-----------------------------------------------------------
"               NERDTree 
"----------------------------------------------------------
"type help: nerdtree for more info
"autocmd vimenter * NERDTree     "自动打开NERDTree
map <F2> :NERDTreeToggle<CR>  "打开和关闭的快捷键
autocmd bufenter * if (winnr("$") == 1 && exists("b:NERDTreeType") && b:NERDTreeType == "primary") | q | endif "自动关闭

"-----------------------------------------------------------
"               Cscope 
"----------------------------------------------------------
if has("cscope")
  set csprg=/usr/bin/cscope
  set csto=1
  set cst
  set nocsverb
  " add any database in current directory
  if filereadable("cscope.out")
     cs add cscope.out
  " else add database pointed to by environment
  elseif $CSCOPE_DB != ""
    cs add $CSCOPE_DB
  endif
  
  set csverb
endif

nmap <C-_>s :cs find s <C-R>=expand("<cword>")<CR><CR>
nmap <C-_>g :cs find g <C-R>=expand("<cword>")<CR><CR>
nmap <C-_>c :cs find c <C-R>=expand("<cword>")<CR><CR>
nmap <C-_>t :cs find t <C-R>=expand("<cword>")<CR><CR>
nmap <C-_>e :cs find e <C-R>=expand("<cword>")<CR><CR>
nmap <C-_>f :cs find f <C-R>=expand("<cfile>")<CR><CR>
nmap <C-_>i :cs find i ^<C-R>=expand("<cfile>")<CR>$<CR>
nmap <C-_>d :cs find d <C-R>=expand("<cword>")<CR><CR>

" Using 'CTRL-spacebar' then a search type makes the vim window
" split horizontally, with search result displayed in
" the new window.

nmap <C-\>s :scs find s <C-R>=expand("<cword>")<CR><CR>
nmap <C-\>g :scs find g <C-R>=expand("<cword>")<CR><CR>
nmap <C-\>c :scs find c <C-R>=expand("<cword>")<CR><CR>
nmap <C-\>t :scs find t <C-R>=expand("<cword>")<CR><CR>
nmap <C-\>e :scs find e <C-R>=expand("<cword>")<CR><CR>
nmap <C-\>f :scs find f <C-R>=expand("<cfile>")<CR><CR>
nmap <C-\>i :scs find i ^<C-R>=expand("<cfile>")<CR>$<CR>
nmap <C-\>d :scs find d <C-R>=expand("<cword>")<CR><CR>
" Hitting CTRL-space *twice* before the search type does a vertical
" split instead of a horizontal one

nmap <C-\><C-\>s
        \:vert scs find s <C-R>=expand("<cword>")<CR><CR>
nmap <C-\><C-\>g
        \:vert scs find g <C-R>=expand("<cword>")<CR><CR>
nmap <C-\><C-\>c
        \:vert scs find c <C-R>=expand("<cword>")<CR><CR>
nmap <C-\><C-\>t
        \:vert scs find t <C-R>=expand("<cword>")<CR><CR>
nmap <C-\><C-\>e
        \:vert scs find e <C-R>=expand("<cword>")<CR><CR>
nmap <C-\><C-\>i
        \:vert scs find i ^<C-R>=expand("<cfile>")<CR>$<CR>
nmap <C-\><C-\>d
        \:vert scs find d <C-R>=expand("<cword>")<CR><CR>

	
"-----------------------------------------------------------
"               color scheme
"                         https://github.com/altercation/vim-colors-solarized
"                         https://github.com/tomasr/molokai
"----------------------------------------------------------

"set background=light
"colorscheme solarized

"set background=dark
"colorscheme solarized

colorscheme molokai
let g:molokai_original = 1

"-----------------------------------------------------------
"               auto complete
"                   https://github.com/vim-scripts/OmniCppComplete
"----------------------------------------------------------

let OmniCpp_MayCompleteDot = 1   " 输入 .  后自动补全
let OmniCpp_MayCompleteArrow = 1 " 输入 -> 后自动补全 
let OmniCpp_MayCompleteScope = 1 " 输入 :: 后自动补全 
let OmniCpp_ShowPrototypeInAbbr = 1 " 显示函数参数列表 


"-----------------------------------------------------------
"               syntastic
"             https://github.com/vim-syntastic/syntastic
"----------------------------------------------------------
set statusline+=%#warningmsg#
set statusline+=%{SyntasticStatuslineFlag()}
set statusline+=%*

let g:syntastic_always_populate_loc_list = 1
let g:syntastic_auto_loc_list = 1
let g:syntastic_check_on_open = 1
let g:syntastic_check_on_wq = 0
```
参考
------
http://www.vim.org/
https://www.zhihu.com/question/26713049/answer/33887444
https://github.com/VundleVim/Vundle.vim/wiki/Examples

  [1]: http://static.zybuluo.com/shenyuflying/cj63bslrkuvv0nidd53wser8/image_1b3rg746h1dpr29dsdu74o1u4qm.png




如果还想看到更多此类文章，请移步到[小宇的博客](http://shenyu.wiki)。

# VIM+TMUX

共享剪切板，复制粘贴???

### 常用命令

```
let mapleader=" "
ciw 				词中删除
ci"					删除引号中的词，进入写入模式
df:					一直删除到冒号

```

```
:e <file>			编辑新文件
```

#### 标签

```shell
set tabpagemax=18 				#VIM默认只能打开10个标签页，在配置文件可以修改这个限制： 
gt, :+tabnext 					#跳转后一个标签页 
gT, -tabnext					#跳转前一个标签页
:tabe filename 					#用标签页打开文件
:tab split 						#用标签页打开当期编辑文件
:tabs 							#显示所有标签页
:tabm 0/1/2 					#跳转到指定标签
:tabfist 						#跳转到第一个标签页
:tablast 						#跳转到最后一个标签页
```

### ctags



### cscope

```shell
find . -name "*.[h|c]" > cscope.files  
cscope -bkq -i cscope.files  
```

### Plugins

#### YCM

```shell
# edit ~/.vimrc
call plug+begin('~/.vim/plugged')
Plug 'Valloric/YouCompleteMe'
call plug#end()
手动进入YCM目录, 输入python3 install.py
let g:ycm_global_ycm_extra_conf='~/.vim/plugged/YouCompleteMe/third_party/ycmd/cpp/ycm/.ycm_extra_conf.py'	#指定.ycm_extra_conf.py路径
.ycm_extra_conf.py内容根据具体系统头文件路径而定
```






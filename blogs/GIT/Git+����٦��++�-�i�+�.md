## Git基础与常用命令

### 0. git帮助

* git help命令

* git原理和命令可参考《Pro Git》，中文版链接:<a href="url"> https://www.progit.cn</a>

### 1. git配置

通过git config命令配置。--global选项指定读写的配置文件路径为~/.gitconfig，只针对当前用户。

```shell
git config --global user.name  "user" 		  #设置用户名
git config --global user.email "user@163.com" #设置邮箱
git config --global core.editor emacs 		  #设置默认文本编译器为emacs
git config --list 							  #检查所有git配置
git config <key> 							  #检查git某一项配置，如user.name 
```

#### 忽略文件 —— .gitignore

* .gitginore文件作用: 忽略无需纳入git管理的文件。

* 各种语言的.gitignore写法可参考: <a href="url">https://github.com/github/gitignore/blob/master/</a>

### 2. 获取与创建git仓

```shell
git init        		#将当前目录初始化为git仓
git clone [url]  		#克隆现有仓库
```

### 3. 添加/删除文件

#### 跟踪文件

```shell
git add <file>	 		#跟踪某个新文件，将内容从工作目录添加到暂存区
git add .		 		#跟踪所有新文件
```

#### 移除文件

``` shell
git rm    				#从git中移除文件，并连带从工作目录中删除指定文件
git rm -f 				#如删除之前有修改并已放到暂存区，必须指定-f选项，防止误删还没有添加到快照的数据不能被git恢复
git rm --cached         #把文件从git暂存区删除，但在工作目录保留该文件
git mv <file1> <file2>  #移动文件，相当于执行以下三条命令
					    #mv file1 file2, git rm file1, git add file2
```

#### 提交更新

```shell
git commit       		#提交更新  
```

### 4. 查看信息

#### 查看当前文件状态

```shell
git status       		#检查当前文件状态
```

#### 查看提交历史

```shell
git log --pretty=oneline #将每个提交放在一行显示
git log -p -2 			 #-p用来显示每次提交的内容差异, -2表示显示最近两次提交
git log --stat 			 #查看每次提交的简略信息
git reflog				 #显示最近的提交记录
```

#### 查看修改和差异

```shell
git diff 					#比较工作目录中当前文件和暂存区快照之间差异，即修改后未暂存的变化
git diff --cached [file] 	#查看暂存区与上一个commit的差异
git diff --staged [file]	#等同于--cached
git diff HEAD				#显示工作区与当前分支最新commit的差异
git diff [commitID1] [commitID2] #比较两次提交记录的差异，比如HEAD和HEAD~1
```

### 5. 分支

#### 查看分支

```shell
git branch								 #列出所有本地分支
git branch -a							 #列出所有本地分支和远程分支
```

#### 新建分支

```shell
git branch [branch_name]				 #创建新分支，但仍然停留在当前分支
git branch [branch_name] [commitID]		 #创建新分支，并指向指定commit
git checkout -b [branch_name] [tag_name] #在特定标签上创建一个分支
```

#### 切换分支

```shell
git checkout [branch_name] 				 #切换分支
git checkout -b [branch_name]			 #创建并切换分支
```

#### 删除分支

```shell
git branch -d [branch_name]				 #删除分支
git branch -D [branch_name]				 #强制删除分支
```

#### 合并分支

```shell
git merge  [branch]			   #将branch分支内容合并到当前分支
git rebase [branch]			   #将branch分支内容变基到当前分支
git rebase [branch1] [branch2] #将branch2分支变基到目标分支branch1，省去切换分支的步骤
```

##### merge和rebase的区别

* merge —— 把两个分支的最新快照及二者最近的共同祖先进行三方合并。
* rebase —— 变基，将提交到某一个分支所有修改移到另一个分支。

<font color ='red'>**注：只对从未推送至公共仓库的提交执行变基命令**</font>，只把变基命令用作推送前清理提交使之整洁的工具。

### 6. 撤销、清理、重写

#### 撤销操作

```shell
git reset HEAD <file> 	 		#取消暂存的文件
git checkout -- <file>   		#撤销文件修改，将文件还原成上次提交的状态
git reset --hard <commitID>		#回退到具体版本号
```

#### 清理工作目录

```shell
git clean -df 				# 移除工作目录中没有忽略的未追踪文件及空的子目录, -d表示删除,-f表示强制
git clean -dn 				# -n查看将会删除哪些文件,不会真正删除
git clean -xdf 	   			# 指定-x额外移除已忽略的未追踪文件
```

#### 重写历史

```shell
git commit --amend  		#修改最后一次提交
git rebase -i [commitID]    #修改多个历史提交
```

### 7. 项目共享与更新

#### 远程仓库

```shell
git remote -v 					 #查看远程仓库
git remote add <shortname> <url> #添加新的远程git仓库
git remote rm <shortname>        #移除远程仓库
```

#### 抓取、推送

``` shell
git fetch [remote] 		 		    #拉取远程仓库数据到本地，但不会自动合并
git pull [remote] [branch]		    #相当于git fetch和git merge命令的组合
git push [remote] [branch]		    #推送本地分支到远程, -f强制推送
git push [remote] –-delete [branch] #删除远程分支
```

### 8. git设置别名

通过git config为git命令创建别名后，无需每次输入完整的git命令，简化了操作

```shell
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.unstage 'reset HEAD --'
git config --global alias.last 'log -1 HEAD'
```

### 参考资料

《Pro Git》第2版： <a href="url">https://www.progit.cn</a>


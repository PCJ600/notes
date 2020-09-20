## git rebase合并本地多次提交记录

#### 操作步骤

**首先用git log命令查看历史提交记录**，示例git仓信息如下：

```shell
$ git log --pretty=oneline
f70c5a84996c05511d3f98034d56ca05706d62f8 (HEAD -> test) fourth commit
56a79bb29da6e483fc6de6e8f271e1a5dcba52a5 third commit
64db6fddd02a04194b3ca22e91dd1de23f9f81d7 second commit
783795e5285155f37c10b72ec9160e554c198ae0 first commit
```

比如这里希望合并本地的前三条记录(f70c, 56a7, 64db)，**需找到待合并记录(64db)的前一条记录的commitID(7837)，作为git rebase -i命令的参数**。

**输入git rebase -i 7837, 进入历史提交的编辑界面：**

![图片2素材](C:\Users\pc\Desktop\图片2素材.png)

需要注意的是，上图显示的提交顺序与git log是相反的。**将除了第一行的pick都改成squash, 保存退出(:wq)**进入commit编辑页面，将commit信息改成merge three commit, 保存退出，再次使用git log查看：

```shell
$ git log --pretty=oneline
f448ac261c7e26682935201244eda0e9d93fd307 (HEAD -> test) merge three commits
783795e5285155f37c10b72ec9160e554c198ae0 first commit
```

发现f70c, 56a7, 64db三条本地记录已经被成功合并为一条新记录。

#### 注意事项

<font color='red'>**只对从未推送至公共仓库的提交记录执行git rebase**</font>

原因可参考<a href = "https://www.progit.cn/">https://www.progit.cn/</a>  "Git分支 -> 变基的风险"，该小节详细讲述了在一个公共仓库执行变基操作的问题。

#### 参考文档

<a href = "https://www.progit.cn/">https://www.progit.cn/  </a>

 














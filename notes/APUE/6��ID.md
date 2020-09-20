### 6个ID

| 实际用户ID | **我是谁，登录UNIX系统时的身份ID** |

| 有效用户ID | **定义操作者的权限，是进程的属性，决定进程对文件的访问权限** |

| 文件所有者ID | **创建文件由进程实现，将该进程的有效ID作为文件所有者ID** |

设置SUID位，则有效用户ID等于文件所有者的uid, 设置SGID位，有效用户ID为文件所有者gid



#### 进一步理解

RUID 实际用户ID： 身份的标识

* 进程能做什么，不是由RUID来决定的

EUID 有效用户ID:    权利的标识

* EUID是进程的权利标识

默认RUID == EUID， 怎么显示?

UID的世袭

* 子进程的RUID == EUID, 继承父进程的RUID

setUID

* 普通用户使用passwd，RUID不变，改变EUID
* chmod 4775 设置 chmod 0775 










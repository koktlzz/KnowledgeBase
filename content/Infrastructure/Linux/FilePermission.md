---
title: "文件权限"
description: ""
date: 2021-02-24T22:05:45+08:00
lastmod: 2021-02-24T22:05:45+08:00
draft: false
images: []
menu:
  infrastructure:
    parent: "Linux"
weight: 900
---

## 默认权限

```bash
[root@koktlzz ~]# mkdir test
[root@koktlzz ~]# ll -d test/
drwxr-xr-x 2 root root 6 4 月   8 10:06 test/
[root@koktlzz ~]# umask 
0022
```

新创建目录的默认权限并非 777，这是因为一些权限被 **umask** 清除了。由于当前 umask 值为 022，因此 test 目录默认被取消了所属组和其他用户的写权限（777-022=755）。

## 特殊权限

除了常见的读、写和执行权限 rwx 外，目录/文件还可能会被赋予一些特殊权限。比如 **passwd** 命令执行的二进制文件：

```bash
[root@koktlzz ~]# ls -l /usr/bin/passwd 
-rwsr-xr-x. 1 root root 33600 4 月   7 2020 /usr/bin/passwd
```

该可执行文件所属用户的权限为 rws，s 即 setuid/setgid，意为该文件会以其所属的用户/组的身份执行，而非运行命令的用户/组。即使普通用户无法修改 **/etc/shadow** 文件，但 **passwd** 命令依然会以 root 用户的身份执行，因此普通用户可以使用该命令修改其密码：

```bash
[root@koktlzz ~]# su kokt
[kokt@koktlzz root]$ passwd
Changing password for user kokt.
Current password: 
```

setgid 权限也可以应用于目录，使目录中新创建的文件加入目录的所属组，而非创建文件用户的所属组。

```bash
[root@koktlzz ~]# usermod -G zz kokt && su kokt
[kokt@koktlzz root]$ umask 
0002
[kokt@koktlzz root]$ mkdir /tmp/shared && chown :zz /tmp/shared/ &&touch /tmp/shared/defaults
[kokt@koktlzz root]$ ll -d /tmp/shared
drwxrwxr-x 2 kokt zz 22 Apr  8 15:12 /tmp/shared
[kokt@koktlzz root]$ ll /tmp/shared/defaults 
-rw-rw-r-- 1 kokt kokt 0 Apr  8 14:50 /tmp/shared/defaults
```

在默认情况下，即使修改目录 shared 的所属组为 zz，用户 kokt 在该目录中创建的文件 defaults 的所属组依然为 kokt。赋予 shared 目录 setgid 权限后，新创建的文件 new 便继承了上级目录的所属组：

```bash
[kokt@koktlzz root]$ chmod g+s /tmp/shared/ && touch /tmp/shared/new 
[kokt@koktlzz root]$ ll /tmp/shared/
total 0
-rw-rw-r-- 1 kokt kokt 0 Apr  8 14:50 defaults
-rw-rw-r-- 1 kokt zz   0 Apr  8 15:05 new
```

还有一种特殊权限名为粘滞位（sticky bit），它可以限制目录内文件的删除。其他用户（Others）有对该目录的读写权限，但只有该目录的所属用户或 root 用户才能删除目录中的文件。例如 tmp 目录权限中的 rwt：

```bash
[root@koktlzz ~]# ll -d /tmp/
drwxrwxrwt. 13 root root 4096 Apr  8 13:49 /tmp/
```

为文件赋予 setuid/setgid 权限的方式与普通权限相同，不过 setuid 仅对可执行文件有效：

```bash
[root@koktlzz ~]# touch test
[root@koktlzz ~]# chmod u+s test 
[root@koktlzz ~]# ll test 
-rwSr--r-- 1 root root 0 Apr  8 13:34 test
[root@koktlzz ~]# chmod u+x test 
[root@koktlzz ~]# ll test 
-rwsr--r-- 1 root root 0 Apr  8 13:34 test
```

为 test 文件添加执行权限后，s 由大写变为小写，这时 setuid 权限才真正生效。同样也可以用数字的方式赋予特殊权限：`setuid = 4, setgid = 2, sticky bit = 1, none = 0`

```bash
[root@koktlzz ~]# chmod u-s test 
[root@koktlzz ~]# ll test 
-rwxr--r-- 1 root root 0 Apr  8 13:34 test
[root@koktlzz ~]# chmod 4744 test 
[root@koktlzz ~]# ll test 
-rwsr--r-- 1 root root 0 Apr  8 13:34 test
```

**chmod** 命令中 4744 的第一位数字便代表为文件赋予 setuid 权限，后三位则分别代表 user、group 和 others 的读、写和执行权限。这也解释了为什么给文件赋予普通权限时的数字常为三位，但 **umask** 命令输出的数字却为四位。

上述三种特殊权限总结如下：

| 特殊权限名称 | 赋予方式 | 赋予文件的效果 | 赋予目录的效果 |
| - | - | - | - |
| setuid | u+s | 文件以其所属用户的身份执行而非运行命令的用户 | 无效果 |
| seigid | g+s | 文件以其所属组的身份执行而非运行命令用户的所属组 | 目录中新创建文件的所属组与目录的所属组相同 |
| sticky bit | o+t | 无效果 | 拥有目录写权限的用户也无法删除或强制保存目录中属于其他用户的文件 |

## ACL 权限（Access Control List）

标准的文件权限只可以对文件所属用户/组和其他用户（user、group 和 others）的使用进行限制。而 ACL 权限则能够对文件权限进行更加细粒度（fine-grained）的划分，它允许为任何用户或用户组设置任何文件的使用权限。

如果使用 **ll** 命令查看文件时发现其权限字符串中末尾有"+"号，那便说明文件被赋予了 ACL 权限：

```bash
[root@koktlzz ~]# ll test 
-rwxrw-r--+ 1 root root 0 Apr 10 10:54 test
```

使用 **getfacl** 命令可以查看其 ACL 权限：

```bash
[root@koktlzz ~]# getfacl test 
# file: test
# owner: root
# group: root
user::rwx
user:kokt:rw-
group::r--
group:zz:rw-
mask::rw-
other::r--
```

每行输出结果的含义如下：

- file/owner/group：文件名/文件所属用户/文件所属组，和标准文件权限相同；
- user::rwx：文件所属用户拥有对该文件的读、写和执行权限；
- user:kokt:rw-：用户 kokt 对该文件拥有读写权限，无法执行；
- group::r--: 文件所属组只拥有对该文件的读权限；
- group:zz:rw-：用户 zz 对该文件拥有读写权限，无法执行；
- mask::rw-：除文件拥有者外，所有的用户/组所能拥有的最大权限为读和写。因此即使将用户 kokt 的权限设置为"rwx"，也无法执行该文件；
- other::r--：其他用户对该文件只拥有读权限。

我们发现上述两条命令的输出结果在文件所属组的权限上出现了偏差：前者为"rw-"，后者则为"r--"。这是因为对于已经赋予了 ACL 权限的文件，其 **ll** 命令输出的权限字符串中的 4～6 位不再是文件所属组的权限了，而是其 ACL 权限中 mask 的值，即"rw-"。1～3 位和 7～9 位的字符串则和标准文件权限一致，分别表示文件所属用户和其他用户的权限。

目录的 ACL 权限和文件相比也有着一些不同，以被赋予了特殊权限 setgid 以及 ACL 权限的 testdir 目录为例：

```bash
[root@koktlzz ~]# ll -d testdir/
drwxrwsr-x+ 2 root root 6 Apr 10 17:12 testdir/
[root@koktlzz ~]# getfacl testdir/
# file: testdir/
# owner: root
# group: root
# flags: -s-
user::rwx
user:kokt:rwx
group::r-x
group:zz:rw-
mask::rwx
other::r-x
default:user::rwx
default:user:kokt:rwx
default:group::r-x
default:group:zz:rw-
default: mask::rwx
default:other::r-x
```

每行输出结果的含义如下：

- file/owner/group：目录名/目录所属用户/目录所属组，和标准文件权限相同；
- flags: -s-：该目录被赋予了 setgid 权限；
- user::rwx/user:kokt:rwx/group::r-x/group:zz:rw-/mask::rwx/other::r-x：与文件的 ACL 权限相同；
- 以 default 开头的各项：目录中的新文件以及子目录将继承的 ACL 权限。

切换到用户 kokt 并在该目录下新建一个文件 test：

```bash
[root@koktlzz ~]# su kokt
[kokt@koktlzz root]$ touch testdir/test && getfacl testdir/test
# file: testdir/test
# owner: kokt
# group: root
user::rw-
user:kokt:rwx                   #effective:rw-
group::r-x                      #effective:r--
group:zz:rw-
mask::rw-
other::r--
```

该文件由用户 kokt 创建，所属者为 kokt。由于上级目录 testdir 被赋予了 setgid 权限，因此 test 文件的所属组为 testdir 的所属组 root 而非创建者的所属组 kokt。虽然用户 kokt 继承了上级目录的 rwx 权限，但受 mask::rw-的限制，实际上只能拥有对该文件的读写权限（#effective:rw-）。

为什么 testdir 目录的 default mask 为 rwx，新文件 test 的 mask 就只有 rw 了呢？这是因为标准的文件权限规定，新创建的文件无法直接获得执行权限，必须手动赋予。因此文件继承的 ACL 权限必须去掉执行权限 x。这也解释了为什么文件所属组的权限为 r-x，但实际生效的只有读权限（#effective:r--）。

如果在 testdir 目录下创建一个子目录 dir，那么该目录的 ACL 权限就和上级目录完全相同了：

```bash
[kokt@koktlzz root]$ mkdir testdir/dir && getfacl testdir/dir
# file: testdir/dir
# owner: kokt
# group: root
# flags: -s-
user::rwx
user:kokt:rwx
group::r-x
group:zz:rw-
mask::rwx
other::r-x
default:user::rwx
default:user:kokt:rwx
default:group::r-x
default:group:zz:rw-
default: mask::rwx
default:other::r-x
```

文件/目录的 ACL 权限赋予基于 **setfacl** 命令，它的主要用法如下：

为某一用户/组赋予某一文件/目录的 ACL 权限：

```bash
setfacl -m u:<username>:<permission rwx> <file/dirname>
setfacl -m g:<groupname>:<permission rwx> <file/dirname>
```

若要同时为目录以及目录中的文件和子目录赋予 ACL 权限，只需添加标识位-R：

```bash
setfacl -R -m u:<username>:rX <dirname>
```

大写 X 是标准文件权限中的一种特殊用法，即只为目录而非文件赋予执行权限。在此处应用可以防止将目录中的一些普通文件被 **setfacl -R** 赋予执行权限，而目录中本来就拥有执行权限的文件则不会受到影响。

给目录的 ACL 权限添加 default 条目，以使目录中的新文件和子目录能够继承目录的 ACL 权限：

```bash
setfacl -m d:u:<username>:<permission rwx> <dirname>
setfacl -m d:g:<groupname>:<permission rwx> <dirname>
```

指定 ACL 权限的 mask：

```bash
setfacl -m m::<permission rwx> <file/dirname>
```

删除文件/目录的 ACL 权限：

```bash
# 删除某一用户/组的 ACL 权限
setfacl -x u:<username> <file/dirname>
setfacl -x g:<groupname> <file/dirname>
# 删除全部 ACL 权限
setfacl -b <file/dirname>
```

将 **getfacl** 命令的输出结果作为输入：

```bash
# fileA 和 fileB 将会拥有相同的 ACL 权限
# --set-file 标识符可以接受来自于文件和 stdin 的输入，此处的-便代表 stdin
getfacl fileA | setfacl --set-file=- fileB
```

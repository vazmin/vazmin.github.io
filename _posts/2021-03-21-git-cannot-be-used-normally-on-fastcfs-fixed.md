---
title: Git 在 FastCFS 上无法正常使用的复现、分析定位（已修复）
date: 2021-03-21 21:11:12 +0800
categories:
  - Analytics
tags:
  - FastCFS Git
toc: true
---


<b>注意： `FastCFS` 已修复该问题 </b>
[fix bug for removing hard link](https://github.com/happyfish100/FastCFS/commit/3e64de743816847b40f748bfc71e27caa4ae78c7)

### 复现

在`FastCFS`下执行`git clone https://github.com/happyfish100/FastCFS.git`

```text
正克隆到 'FastCFS'...
remote: Enumerating objects: 19, done.
remote: Counting objects: 100% (19/19), done.
remote: Compressing objects: 100% (19/19), done.
remote: Total 1129 (delta 3), reused 4 (delta 0), pack-reused 1110
接收对象中: 100% (1129/1129), 1.68 MiB | 490.00 KiB/s, done.
处理 delta 中: 100% (701/701), done.
error: wrong index v1 file size in /usr/local/fastcfs-test-standalone/fuse/fuse1/FastCFS/.git/objects/pack/pack-3d4a9f6c578cb3a72a4b2526c0af410fb34ea163.idx
error: wrong index v1 file size in /usr/local/fastcfs-test-standalone/fuse/fuse1/FastCFS/.git/objects/pack/pack-3d4a9f6c578cb3a72a4b2526c0af410fb34ea163.idx
error: wrong index v1 file size in /usr/local/fastcfs-test-standalone/fuse/fuse1/FastCFS/.git/objects/pack/pack-3d4a9f6c578cb3a72a4b2526c0af410fb34ea163.idx
fatal: bad object f3a15bb4acefb676463b92e6d5db756fb14cde93
fatal: 远程没有发送所有必须的对象
Unexpected end of command stream
```

尝试 `git` 建仓、添加文件、提交文件

```shell
 git init foo && cd foo
 echo 'test content' > foo.txt
 git commit -m 'test'
```

此时`commit`失败

```
error: unable to unpack 7cb102b7d7bfe665f816404583b575cdb883e87b header
error: inflateEnd: stream consistency error (no message)
fatal: 7cb102b7d7bfe665f816404583b575cdb883e87b is not a valid object
```

### 分析

结合报错日志`wrong index v1 file size`, 初步认为是文件元数据出了问题。关闭FastDIR的异步报告将fuse.conf的`async_report_enabled`设置为`false`。
```
# if async report file attributes (size, modify time etc.) to the FastDIR server
# default value is true
async_report_enabled = false
```
关闭重启FastCFS服务，问题仍然存在。

开始查找git的文档理解git内部原理。阅读
[10.2 Git 内部原理 - Git 对象](https://git-scm.com/book/zh/v2/Git-%E5%86%85%E9%83%A8%E5%8E%9F%E7%90%86-Git-%E5%AF%B9%E8%B1%A1)后，开始试探。

```
echo 'test content' | git hash-object -w --stdin
# d670460b4b4aece5915caf5c68d12f560a9fe3e4

find .git/objects -type f
# .git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4

git cat-file -p d670460b4b4aece5915caf5c68d12f560a9fe3e4
# error: unable to unpack d670460b4b4aece5915caf5c68d12f560a9fe3e4 header
# error: inflateEnd: stream consistency error (no message)
# fatal: Not a valid object name d670460b4b4aece5915caf5c68d12f560a9fe3e4
```

将在本地文件系统执行一样的操作得到`.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4`与`FastCFS`的`objetcs`对象对比。

```shell
file .git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4
# FastCFS: .git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4: data
# Local FS: .git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4: VAX COFF executable not stripped - version 737

hexdump .git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4
# FastCFS
# 0000000 0000 0000 0000 0000 0000 0000 0000 0000
# 0000010 0000 0000 0000 0000 0000 0000 0000
# 000001d
# Local FS
# 0000000 0178 ca4b 4fc9 3052 6634 4928 2e2d 4851
# 0000010 cfce 492b 2bcd 02e1 4b00 07df 0009
# 000001d
```

明显两个object文件不一致。此时陷入沉思。将此问题反馈给`@happyfish100`。

### 回顾性分析

用`echo 'test content' | git hash-object -w --stdin`创建一个数据对象并将它手动存到git数据库，
`test content`的`hash-object`为`d670460b4b4aece5915caf5c68d12f560a9fe3e4`,
所以`Git`会在`.git/object`目录下创建`d6`,`d6`下创建文件`70460b4b4aece5915caf5c68d12f560a9fe3e4`[<sup>1</sup>](#refer-anchor-1)。 
同时查看`FastDIR`的`binlog`
```
0178<rec dv=815 id=9007199258741264 op=2,cr ts=1616335625 ns=2,fs pt=9007199258741261 nm=2,d6 hc=3277 md=16877 bt=1616335625 at=1616335625 ct=1616335625 mt=1616335625 ui=0 gi=0/rec>

0191<rec dv=816 id=9007199258741265 op=2,cr ts=1616335625 ns=2,fs pt=9007199258741264 nm=14,tmp_obj_IpJax6 hc=3277 md=33060 bt=1616335625 at=1616335625 ct=1616335625 mt=1616335625 ui=0 gi=0/rec>

0085<rec dv=817 id=9007199258741265 op=2,up ts=1616335625 hc=3277 sz=29 se=29 ia=29/rec>

0237<rec dv=818 id=9007199258741266 op=2,cr ts=1616335625 ns=2,fs pt=9007199258741264 nm=38,70460b4b4aece5915caf5c68d12f560a9fe3e4 hc=3277 md=4227583 bt=1616335625 at=1616335625 ct=1616335625 mt=1616335625 ui=0 gi=0 si=9007199258741265/rec>

0116<rec dv=819 id=9007199258741265 op=2,rm ts=1616335625 ns=2,fs pt=9007199258741264 nm=14,tmp_obj_IpJax6 hc=3277/rec>
```
即动作如下

dv 815: create `d6`

dv 816: create `tmp_obj_IpJax6`

dv 817: up

dv 818: create `70460b4b4aece5915caf5c68d12f560a9fe3e4` (由`tmp_obj_IpJax6`硬链接的生成？)

dv 819: rm tmp_obj_IpJax6

此时猜测git是先创建临时文件`tmp_obj_IpJax6`并写入内容, 
然后创建`Hard Link` 文件`70460b4b4aece5915caf5c68d12f560a9fe3e4`， 
最后删除 `tmp_obj_IpJax6`。

### 验证

```shell
echo 'test content' > foo.txt
ln foo.txt bar.txt
rm foo.txt
```
正常情况下`cat bar.txt`应该输出内容`test content`，但在`FastCFS`下没有。

### 结论

`FastCFS` 没有正确处理 `Hard Link` 导致 git 不是正常使用。当然，`FastCFS`的作者很快就修复了这个问题。


### 参考

<div id="refer-anchor-1"></div>
- [1] [10.2 Git 内部原理 - Git 对象](https://git-scm.com/book/zh/v2/Git-%E5%86%85%E9%83%A8%E5%8E%9F%E7%90%86-Git-%E5%AF%B9%E8%B1%A1)
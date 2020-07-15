### 一、仓库复制

环境

| 服务器       |                |      |
| ------------ | -------------- | ---- |
| 192.168.0.43 | 镜像仓库服务器 |      |
| 192.168.0.66 | 源库服务器     |      |



#### 1.1  创建镜像仓库

在镜像服务器上执行

```bash
svnadmin create /u01/svndata/svn/root
```



#### 1.2 设置钩子脚本

##### 1.2.1 编辑pre-revprop-change

只允许syncuser修改SVN版本属性

/var/svn/svn-mirror/hooks/pre-revprop-change

```bash
#!/bin/sh 

USER="$3"

if [ "$USER" = "syncuser" ]; then exit 0; fi

echo "Only the syncuser user may change revision properties" >&2
exit 1
```



##### 1.2.2 编辑 start-commit 钩子脚本

只允许用户 `syncuser` 向仓库提交新的版本号

```bash
#!/bin/sh 

USER="$2"

if [ "$USER" = "syncuser" ]; then exit 0; fi

echo "Only the syncuser user may commit new revisions" >&2
exit 1
```



##### 1.2.3  授权可执行给钩子脚本

```bash
chmod u+x pre-revprop-change
chmod u+x start-commit
```



#### 1.3 初始化SVN版本库

在源库服务器上执行

```bash
svnsync initialize http://192.168.0.43/svn/root \
                     http://192.168.0.66/svn/root \
                     --sync-username syncuser --sync-password syncpass
```



#### 1.4 复制仓库到镜像

*方案一，从0开始同步*

请求源仓库的服务器开始重放从 0 到最新版本号之间的所有版本号. 随着 *svnsync* 不断地从源仓库服务器获取到结果, 它开始把版本号作为新提交, 转发到目标仓库服务器上.

```bash
 svnsync synchronize http://192.168.0.43/svn/root \
                      http://192.168.0.66/svn/root
```



*方案二， 直接复制仓库, 然后把复制出的仓库初始化成新镜像*

```bash
svnadmin hotcopy /path/to/repos /path/to/mirror-repos
svnsync initialize --allow-non-empty file:///path/to/mirror-repos \
                                       file:///path/to/repos
```



### 二、 版本合并

#### 2.1 新特性分支策略

> 进行一个长期的新特性开发，需要建立一个分支和主干的管理流程



##### 2.1.1 建立特性分支

```bash
svn copy ^/calc/trunk ^/calc/branches/my-calc-branch \
           -m "Creating a private branch of /calc/trunk."
```



##### 2.1.2 保持分支同步

经常保持分支与开发主线同步可以降低分支被合并到主干上时发生冲突的概率.

```bash
cd /home/user/my-calc-branch
svn up
svn merge ^/calc/trunk
```



##### 2.1.3 处理可能产生的冲突

 没有语法冲突并不表 示没有语义冲突,所以还是要进行代码构建测试

如果合并后产生了很多问题,可以撤消本地的所有修改

```bash
svn revert . -R
```



##### 2.1.4 提交合并的变更集到特性分支

```bash
svn commit -m "Sync latest trunk changes to my-calc-branch."
```



##### 2.1.5 重新整合分支,再合并

完成新特性开发后，我们需要重新合并一次主干上的新内容，然后在将特性分支合并到主干

```bash
cd /home/user/my-calc-branch
svn up
svn merge ^/calc/trunk
svn commit -m "Final merge of trunk changes to my-calc-branch."
```

分支合并到主干

```bash
cd /home/user/calc-trunk
svn merge ^/calc/branches/my-calc-branch
svn commit -m "Merge my-calc-branch back into trunk!"
```



##### 2.1.6 删除特性分支

```bash
svn delete ^/calc/branches/my-calc-branch \
             -m "Remove my-calc-branch, reintegrated with trunk in r381."
```



#### 2.2 查看合并信息

与合并相关的数据 记录在文件和目录的 `svn:mergeinfo` 属性中

查看分支的合并信息

```bash
$ cd my-calc-branch

# 添加-R可以看见子目录合并信息
$ svn pg svn:mergeinfo -vR
Properties on '.':
  svn:mergeinfo
    /calc/trunk:341-378
```



#### 2.3  查看两个分支间的合并关系

查看目录吸收了哪些变更集, 或者查看哪些变更集它是有资格吸收的

##### 2.3.1 查看分支之间的图形化概览

```bash
$ cd my-calc-branch

$ svn mergeinfo ^/calc/trunk   
    youngest common ancestor
    |         last full merge
    |         |        tip of branch
    |         |        |         repository path

    340                382
    |                  |
  -------| |------------         calc/trunk
     \          /
      \        /
       --| |------------         calc/branches/my-calc-branch
              |        |
              379      382
```



##### 2.3.2 查看分支上合并了哪些版本号

```bash
svn mergeinfo ^/calc/trunk --show-revs merged
```



##### 2.3.3 看分支可以从主干上合并哪些修改

```bash
svn mergeinfo ^/calc/trunk --show-revs eligible
```



#### 2.4 撤销修改

用 *svn merge* 在工作副本中 “撤消” 版本号 392 的修改, 然后把用于撤消 r392 的修改提交到仓库中.

```bash
# 也可以指定参数 --revision 392:391 或 --change -392
svn merge ^/calc/trunk . -c-392
svn commit -m "Undoing erroneous change committed in r392."
```



#### 2.5 恢复已经删除的文件

将文件版本号为399恢复

方法一：*svn log* 将会遍历到 r399 之前的历史

```
svn copy ^/calc/trunk/src/real.c@399 ^/calc/trunk/src/real.c \
           -m "Resurrect real.c from revision 399."
```

方法二，不保留历史

```bash
$ svn cat ^/calc/trunk/src/real.c@399 > ./real.c

$ svn add real.c
A         real.c

# Commit the resurrection.
```



#### 2.6 精选合并（*cherrypicking*）

```bash
 svn merge ^/calc/trunk -c413
 svn merge -r 100:200 http://svn.example.com/repos/trunk
 svn merge -r 100:200 http://svn.example.com/repos/trunk my-working-copy
 svn merge http://svn.example.com/repos/branch1@150 \
            http://svn.example.com/repos/branch2@212 \
            my-working-copy
```






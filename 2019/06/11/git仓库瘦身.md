# git仓库瘦身

### 问题

  在git使用过程中，错误的提交了某些大文件或不必要的文件（如视频文件、node_modules、jar等）,我们意识到提交错误，删除了这些文件并重新提交，但是项目中的.git文件却变得十分臃肿，导致项目clone时耗费大量时间。

### 原因

  当用户提交一个新文件a.mp4(50MB)到仓库,仓库中增加50M,同时git记录中也会保存这个文件，以实现代码的回滚，所以仓库的大小会增加100M（2倍左右）。后来我们发现不需要这个视频文件，删除此文件并提交时，实际上只删除了仓库中的文件，记录中依旧保留。所以.git文件夹会永久保存这个记录，从而造成项目体积变大。
### 解决思路

  要想完全的从你的仓库中删除这个文件，你必须：

  - 从你的项目的当前的文件树中删除该文件;

  - 从仓库的历史记录中删除文件——_重写_ Git 历史记录，从包含该文件的所有的提交中删除这个文件;

  - 删除指向_旧的_提交历史记录的所有 reflog 历史记录;

  - 重新整理仓库，使用 git gc 对现在没有使用的数据进行垃圾回收。

### 注意事项
  > 仓库维护十分危险

  > 务必备份好仓库
  
  > 建议在项目完成一个版本更新，并且与团队成员协商暂停开发后，进行此次清理


### 解决方法
#### 方法一 使用 BFG 重写历史记录
1. 安装java

    ```
      brew cask install java
    ```

2. 创建备份仓库
    ```
    $ git clone --mirror git://example.com/some-big-repo.git
    ```

3.  [下载jar包](https://search.maven.org/classic/remote_content?g=com.madgag&a=bfg&v=LATEST) 放入备份仓库的同级目录 

4. 删除文件记录
    ```
    # 删除大于100M的文件

    $ java -jar bfg.jar --strip-blobs-than 100M some-big-repo.git

    # 删除匹配的文件

    $ java -jar bfg.jar --delete-files *.mp4 some-big-repo.git

    # 删除指定的文件

    $ java -jar bfg.jar --delete-files eg.sdf some-big-repo.git

    # 删除指定的文件夹

    $ java -jar bfg.jar --delete-folders dist some-big-repo.git
    ```

5. 对不用的数据垃圾回收

    - 进入对应仓库
    - 删除从现在到后面的所有 reflog 引用（除非你明确地只在一个分支上操作）。
    - 通过运行垃圾回收器和删除旧的对象重新打包仓库。
    - 把你所有的修改推送回仓库。

    ```
    cd some-big-repo.git/

    git reflog expire --expire=now --all
    git gc --prune=now
    git push --mirror
    ```
6. 现在再次clone仓库，发现仓库变小很多

#### 方法二 使用 git filter-branch 来重写历史记录 【[转](https://www.zhihu.com/question/29769130/answer/315745139)】

1. 运行 gc ，生成 pack 文件（后面的 --prune=now 表示对之前的所有提交做修剪，有的时候仅仅 gc 一下.git 文件就会小很多）
```
$ git gc --prune=now
```
2. 找出最大的三个文件（看自己需要）
```
$ git verify-pack -v .git/objects/pack/*.idx | sort -k 3 -n | tail -3
```
```
# 示例输出：
#1debc758cf31a649c2fc5b0c59ea1b7f01416636 blob   4925660 3655422 14351
#c43a8da9476f97e84b52e0b34034f8c2d93b4d90 blob   154188651 152549294 12546842
#2272096493d061489349e0a312df00dcd0ec19a2 blob   155414465 153754005 
1650961363. 
```
查看那些大文件究竟是谁（c43a8da 是上面大文件的hash码）
```
$ git rev-list --objects --all | grep c43a8da
# c43a8da9476f97e84b52e0b34034f8c2d93b4d90 data/bigfile4.移除对该文件的引用（也就是 data/bigfile）
```
```
$ git filter-branch --force --index-filter "git rm --cached --ignore-unmatch 'data/bigfile'"  --prune-empty --tag-name-filter cat -- --all5.进行 repack $ git for-each-ref --format='delete %(refname)' refs/original | git update-ref --stdin
$ git reflog expire --expire=now --all
$ git gc --prune=now
```
6.查看 pack 的空间使用情况$ git count-objects -v

### 参考

> [使用BFG清除git仓库中的隐私文件或大文件](https://www.cnblogs.com/huipengly/p/8424096.html)

> [git瘦身](http://palanceli.com/2017/12/18/2017/1218GitReduceSize/)

> [BFG Repo-Cleaner](https://rtyley.github.io/bfg-repo-cleaner/)

> [如何瘦身 Git 仓库](https://www.imooc.com/article/71565)

> [如何解决 GitHub 提交次数过多 .git 文件过大的问题？](
https://www.zhihu.com/question/29769130/answer/315745139)

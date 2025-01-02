## Git 学习笔记

### 1. 设置配置信息

* **邮箱密码**

  	git config --global user.password xxx
		
  	git config --global user.email xxx

* **http || https**
  ``` git config --global http.proxy 127.0.0.1:xxx```
  ``` git config --global https.proxy 127.0.0.1:xxx```

* **sock5代理**
   ```git config --global http.proxy socks5 127.0.0.1:xxx```
   ```git config --global https.proxy socks5 127.0.0.1:xxx```

* **查看代理**
* ```git config --global --get http.proxy```
  ```	git config --global --get https.proxy```

* **取消代理**

  	git config --global --unset http.proxy
  	git config --global --unset https.proxy

* **遍历配置**

  ```git config --list```

### 2.  git 命令运用细节

* ```git add file.txt```                        表示将文件更改内容加入暂存区

* ```git status```                                  查看文件状态判断是否在暂存区

* ```git restore --staged file.txt``` 表示将暂存区里的内容清除将状态变成上次提交状态 (注意如果你还未提交过该命令会报错: **fatal: could not resolve HEAD**, 此时你可以使用 git reset 重置成初始版本也就是最开始暂存区为空的时候, 此时你可以再次```git add .```)

* ```git rm --cached file.txt```          将文件取消管理

* ```git restore file.txt```                 将工作区的内容回滚到与暂存区一致

* ```git reset``` 命令

  ```
  1. git reset  默认是返回上一个提交版本
  2. git reset HEAD ^ 返回上一个提交版本
  3. git reset HEAD^ hello.php   回退 hello.php 文件的版本到上一个版本 
  4. git reset 052e      回退到指定版本 
  ```

  **--hard** 参数撤销工作区中所有未提交的修改内容，将暂存区与工作区都回到上一次版本，并删除之前的所有信息提交：

  ```
  git reset --hard HEAD
  ```

  实例：

  ```
  $ git reset --hard HEAD~3  # 回退上上上一个版本  
  $ git reset –hard bae128  # 回退到某个版本回退点之前的所有信息。 
  $ git reset --hard origin/master    # 将本地的状态回退到和远程的一样 
  ```

​	**注意：**谨慎使用 **–-hard** 参数，它会删除回退点之前的所有信息。

* ```git log```                                          查看提交日志

* ```git reflog```                                     提交日志记录head指针的过程

* ```git pull  ```                                         其实就是 **git fetch** 和 **git merge** 的简写，先从远程仓库获取最新的提交记录，然后将这些提交记录合并到你**当前的分支**中。

  ```
  git pull [远程仓库名] [分支名]
  ```

  - `[远程仓库名]` 通常是 `origin`，是默认的远程仓库名。
  - `[分支名]` 是你要合并的远程分支，比如 `main` 或 `master`。
  - ```git branch --set-upstream-to=origin/xxx1 xxx``` 本地仓库没有远程仓库xxx1的时候需要先将远程仓库的xxx1分支与本地仓库xxx分支关联起来
  - 注意一般在协作开发的时候commit之后要进行pull 然后在push, 解决冲突避免push不上去
  
* ``` git branch```                                          查询当前的所有分支

* ```git branch xxx ```                                   创建xxx分支

* ``` git branch -d xxx```                              删除xxx分支

* ```git checkout xxx ```                               切换到xxx分支 

* ```git checkout -b xxx ```                          创建并切换到xxx分支

* ```git push -d origin xxx```                     删除远程仓库 xxx分支

* ```git push --set-upstream origin xxx``` 当远程仓库没有xxx分支的时候需要先将当前分支与远程仓库分支关联起来

* ```git stash```                                            将工作区和暂存区中尚未提交的修改存入栈中

* ```git stash apply```                                  将栈顶存储的修改恢复到当前分支，但不删除栈顶元素

* ```git stash drop```                                    删除栈顶存储的修改

  * ```git stash pop```                                      将栈顶存储的修改恢复到当前分支，同时删除栈顶元素

* ```git stash list```                                     查看栈中所有元素

### 3.错误汇总

* **fatal: The current branch dev1 has no upstream branch.**
  **To push the current branch and set the remote as upstream, use**

  **git push --set-upstream origin dev1**

  表明远程仓库中没有包含这个dev1分支需要你执行 ```git push --set-upstream origin dev1``` 来在远程仓库创建这个dev1分支与本地对应起来

* 
---
layout: post
title:  "Git多库同步管理"
date:   2020-02-16 22:16:01 +0800
categories: 版本控制软件
tag: Git
---

* content
{:toc}

> 翻译

场景:一个项目同时连接两个服务器

## 设置Git

1. 我们要做的第一件事是在命令行上使用以下命令来克隆存储库：

    ```bash
    $ git clone git@github.com:MIT-DB-Class/simple-db-hw.git
    ```

    将工作路径更改为新克隆的存储库：

    ```bash
    $ cd simple-db-hw/
    ```

2. 默认情况下，远程调用的`origin`设置为你从中克隆的存储库的位置。你应该看到以下内容：

    ```bash
    $ git remote -v
        origin git@github.com:MIT-DB-Class/simple-db-hw.git (fetch)
        origin git@github.com:MIT-DB-Class/simple-db-hw.git (push)
    ```

    我们不希望该origin成为源。所以，我们要更改它，指向你的存储库。执行以下命令：

    ```bash
    $ git remote rename origin upstream
    ```

    ```bash
    $ git remote -v
        upstream git@github.com:MIT-DB-Class/simple-db-hw.git (fetch)
        upstream git@github.com:MIT-DB-Class/simple-db-hw.git (push)
    ```

3. 最后，我们需要给你的存储库一个新的`origin`。执行以下命令，替换url：

    ```bash
    $ git remote add origin git@github.com:MIT-DB-Class/homework-solns-2018-<athena-username>.git
    ```

    如果出现以下错误：

    ```
    Could not rename config section 'remote.[old name]' to 'remote.[new name]'
    ```

    或:
   
    ```
    fatal: remote origin already exists.
    ```

    发生这种情况，具体取决于他们使用的Git版本。要解决此问题，只需执行以下命令：

    ```bash
    $ git remote set-url origin git@github.com:MIT-DB-Class/homework-solns-2018-<athena username>.git
    ```

    作为参考，正确设置后的最终执行`git remote -v`应该是如下所示：

    ```bash
    $ git remote -v
        upstream git@github.com:MIT-DB-Class/simple-db-hw.git (fetch)
        upstream git@github.com:MIT-DB-Class/simple-db-hw.git (push)
        origin git@github.com:MIT-DB-Class/homework-solns-2018-<athena username>.git (fetch)
        origin git@github.com:MIT-DB-Class/homework-solns-2018-<athena username>.git (push)
    ```

4. 让我们通过执行以下命令将master分支推送到GitHub进行测试：

    ```bash
    $ git push -u origin master
    ```

    你应该能看到类似以下的内容：

    ```
	Counting objects: 59, done.
	Delta compression using up to 4 threads.
	Compressing objects: 100% (53/53), done.
	Writing objects: 100% (59/59), 420.46 KiB | 0 bytes/s, done.
	Total 59 (delta 2), reused 59 (delta 2)
	remote: Resolving deltas: 100% (2/2), done.
	To git@github.com:MIT-DB-Class/homework-solns-2018-<athena username>.git
	 * [new branch]      master -> master
	Branch master set up to track remote branch master from origin.
    ```

5. 最后一个命令有点特殊，只需要第一次运行就可以设置远程跟踪分支。现在我们应该能够只运行不带参数的`git push`了。试试看，你将获得以下信息：

    ```bash
    $ git push
      Everything up-to-date
    ```

## 获取更新

获取其中一个仓库的更新

```bash
$ git pull upstream master
```

或者，如果你希望更明确，则可以先获取然后合并：

```bash
$ git fetch upstream
$ git merge upstream/master
```
现在提交到你的master分支：
```bash
$ git push origin master
```

[Git学习平台](https://learngitbranching.js.org/)

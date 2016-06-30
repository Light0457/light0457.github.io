---
title: 修正git中错误的提交者姓名
tag: git
---
# 情景  
在一个仓库上提交时，用户名和邮箱给配置错了，看记录才知道。这可不能接受，看着可不舒服。幸好[github帮助页](https://help.github.com/articles/changing-author-info/)（此帮助页有[相对应的中文版本](http://dangzhiqiang.blog.51cto.com/7961271/1657864)）上有提供这种情况下的方法。   
帮助页有下面的警告，对于公有仓库最好不去干这种改记录的事。不过我这是私有仓库就无所谓了。

    Warning: This action is destructive to your repository's history. If you're collaborating on a repository with others, it's considered bad practice to rewrite published history. You should only do this in an emergency.

<!--more-->
# 开始重写历史
1. 按帮助页上所写，需要先clone一个**裸版本库**([bare repository](https://segmentfault.com/q/1010000004683286))。查不到官方对裸版本库的说明或者定义，从搜索到的其它资料看，裸版本库就是一个只带历史信息文件的库，库的工作目录和文件并不会存在于目录中，只存在于历史中。远端共享的版本库经常就是裸版本库（可能是因为没检出工作目录省了空间？）。裸版本库还有其它方面未知的，后面找时间再探讨探讨。对于现在来说，我要修改历史信息的私人仓库来说完全是没这必要特意克隆一个裸版本库出来。不过为了试验修改完历史记录后，其它的fork或者clone是什么样合并这个历史记录的，还是跟着clone一个裸版本库出来好了。

        git clone --bare https://github.com/user/repo.git

2. 帮助页里面提供了一个shell脚本用来批量修改历史记录。如下：

        #!/bin/sh

        git filter-branch --env-filter '
        OLD_EMAIL="your-old-email@example.com"
        CORRECT_NAME="Your Correct Name"
        CORRECT_EMAIL="your-correct-email@example.com"
        if [ "$GIT_COMMITTER_EMAIL" = "$OLD_EMAIL" ]
        then
            export GIT_COMMITTER_NAME="$CORRECT_NAME"
            export GIT_COMMITTER_EMAIL="$CORRECT_EMAIL"
        fi
        if [ "$GIT_AUTHOR_EMAIL" = "$OLD_EMAIL" ]
        then
            export GIT_AUTHOR_NAME="$CORRECT_NAME"
            export GIT_AUTHOR_EMAIL="$CORRECT_EMAIL"
        fi
        ' --tag-name-filter cat -- --branches --tags

    其中filter-branch这个命令在pro-git一书中可是把它标为核弹级选项，同时也在那书中看到了相类似的需要修改提交者姓名的需求和做法。至于两者的具体差异，后面可以找时间再研究下。

3. 修改完后推送上去。到于后面的参数以及为何要加**'refs/heads/*'**这个，等后面有时间再看看吧。

        git push --force --tags origin 'refs/heads/*'

4. 强推时，如果远端分支受保护的话是强推不了的，会遇到下面的错误：

        You are not allowed to force push code to a protected branch on this project.
    
    gitlab上默认是设置master分支是不允许强推的,需要到Project -> Protected Branches -> 把master分支点掉

5. 让其它人拉取。帮助页在前面时有这么段话：  
    
        Note: Running this script rewrites history for all repository collaborators. After completing these steps, any person with forks or clones must fetch the rewritten history and rebase any local changes into the rewritten history.
    
    也就是其它已经Clone过这仓库的人拉取时要用rebase的方式（即将他们本地的修改历史逐个应用到这个新的历史树上）。否则的话，使用merge的方式将会将他们本地的历史记录（那个包含了错误名字的历史记录）重新合并到分支的历史中，并且又把那些错误名字的提交也合并了过来！！！  
    所以其它人拉取时，要切记使用   
        git pull --rebase <remote name> <branch name>
    来拉取。
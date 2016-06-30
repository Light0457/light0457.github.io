---
title: SSH使用杂记
tag: ssh
---
# [SSH网络协议](https://zh.wikipedia.org/wiki/Secure_Shell)使用杂记
最近寻觅各git托管服务，都选择了用ssh协议来拉取。以及最近因为各种原因接触得比较多，碰到的问题也多了，在这里面记录一下。  
* 使用
        ssh-keygen -t rsa -C "youremail-address"
  创建ssh密钥时，
  1. 要求输入生成文件的目录，此时输入的目录需要是完整路径（也不能用～表示Home目录） 
  2. 要求输入passphrase,这个从目前的用法看就是即将生成的密钥文件的密码，把它添加到ssh agnet的时候需要输入这个密码

<!--more-->

* 在ssh-add密钥后，碰到个问题是重启后还要再添加一遍。
        ssh-add ~/.ssh/xxx_rsa  
  这个原因在于ssh-add添加密钥文件，只是用于添加指定文件到当前运行中的ssh agent的session中，重启后session重建，添加的密钥文件也就被重置了（似乎如果创建的密钥文件是默认名id_rsa，则不会有这个问题。可能重启后会添加默认的密钥文件吧）。   
  解决办法有两个：  
  1. 在Mac OS X上，系统自带有keychain access这个管理工具，添加密钥文件可以把密钥文件一起添加到keychain access中，方法是ssh-add时多带一个-K参数   

         ssh-add -K ~/.ssh/xxx_rsa
  
  2. 在~/.ssh中添加config文件，添加对应网站要使用的密钥文件  

         Host gitlab.com  
         RSAAuthentication yes 
         IdentityFile ~/.ssh/xxx_rsa


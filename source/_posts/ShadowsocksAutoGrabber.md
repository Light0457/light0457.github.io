---
title: 科学上网便捷小工具
tags: 
- Shadowsocks     
- plist   
- Alfred    
- python   
---
# 一、概述  

为啥写这篇文章呢～～主要是想记录下一些琐碎的点。最近折腾“科学上网”查到的信息点太琐碎了，实在构不成完整的知识体系，但相关资料不记下来又可惜了点，没准以后还用到。写篇文章来记最容易检索到了。   
最近一直用[ShadowsocksX][ShadowsocksXDownload]来上谷歌，服务器用的是从[iShadowsocks][iShadowsocks]网站获取的免费服务器。无耐何它每六个小时更换一次密码。  
<!--more-->
在老老实实手动更新密码一大段时间后，让我知道了mac上的这个软件：[Alfred](https://www.alfredapp.com/)这个软件，这个软件可以方便地运行脚本、启动程序等等小功能。就是它的这些功能使我萌发了写个脚本自动更新服务器密码的想法。才有了这篇文章。

# 二、抓取服务器地址信息   
个人偏好使用Python写这一类的小工具，库多，写着方便，所以下面的脚本大部分都是用Python写的。  
## 2.1 在Python中访问网站[iShadowsocks][iShadowsocks]  
要抓取服务器地址信息嘛，访问网站在把网页内容下下来是必须的。在Python中做这点倒是挺简单的，像下面一小段代码就好了。

    import urllib2
    html_content = urllib2.urlopen("http://www.ishadowsocks.net/").read()

## 2.2 解析网页内容获取服务器地址
python中解析html网页内容的库有好几个，有[HTMLParser](https://docs.python.org/2/library/htmlparser.html),[HtmlLib](https://docs.python.org/2/library/htmllib.html),[BeautifulSoup](https://www.crummy.com/software/BeautifulSoup/bs4/doc/)。  
* **HtmlLib**: 已经被官方抛弃掉不建议使用了，不去考虑。  
* **HTMLParser**: 看了下HTMLParser的使用例子，需要写个解析类继承于HTMLParser。这种方式并不喜欢，因为类继承不方便组织代码，同时还得看需要重载什么函数，重载的函数负责什么职责。
* **BeautifulSoup**: 对比了下BeautifulSoup，用起来直接是按键值查找，有种将html文件当json来用的感觉，挺好的，写起来省时省力些。  

所以最后用了BeautifulSoup这个库来解析html。  
打开谷歌浏览器看了下该网站的网页代码，虽然我对html并不熟悉，但那个网页还是挺简洁的，还带了良好的溈，看了一下，
1. 我们需要的服务器地址的大概位置就是在一个tag是section,id是free的一个容器里面,  
2. 每个服务器地址的内容使用了一样的div样式，
3. 服务器地址的每一项内容都是用h4来显示的   

总的来说，内容要挺有规律的，要抓取出来还是比较轻松的。   

    soup = BeautifulSoup(html_content)
    free_part = soup.find("section",id = "free")
    free_account_parts = free_part.find_all("div",class_ = "col-lg-4 text-center")
    key_parsers = {
        "server": lambda str : u"服务器地址" in str,
        "server_port": lambda str : u"端口" in str,
        "password": lambda str : u"密码" in str,
        "method": lambda str : u"加密方式" in str
    }
    free_server_infos = []  //** 存储最终抓取到的结果
    for free_account_part in free_account_parts:             
        account_infos = free_account_part.find_all("h4")
        for account_info in account_infos:
            server_info = {}
            values = account_info.text.split(u':')
            if len(values) <= 1:
                continue
            for (key, parser) in key_parsers.items():
                if parser(values[0]):
                    server_info[key] = values[1]
            free_server_infos.append(server_info)

# 三、更新ShadowsocksX的配置
ShadowsocksX这个软件呢用着还是挺方便的，无奈我一直找不到它配置的地方。谷歌后在[它的github issue页面](https://github.com/shadowsocks/shadowsocks-iOS/issues/150)发现，Mac下的这个ShadowsocksX软件并没有像其它版本的Shadowsocks一样提供一个可配置的json文件，而使用了苹果的plist文件来存储。它的配置文件就是这个：~/Library/Preferences/clowwindy.ShadowsocksX.plist。
## 3.1 使用Python读写plist配置文件
python读写plist的库我是搜到了两个,一个是[plistlib](https://docs.python.org/2/library/plistlib.html),另一个则是[biplist](https://pypi.python.org/pypi/biplist/1.0.1),两个库接口用起来都差不多。由于plistlib是官方库，刚开始选了它。但拿它来读其它plist文件很正常，拿来读clowwindy.ShadowsocksX.plist这个文件却会报错。而用biplist却不会。因此最终选择了biplist做为这次读写plist文件的python库。   
读写plist文件还是很简单，几句代码就好了

    //** 获取当前用户下的clowwindy.ShadowsocksX.plist路径
    import os
    plist_file_path = os.path.join(os.path.expandvars('$HOME'),"Library/Preferences/clowwindy.ShadowsocksX.plist")

    import biplist
    plist = biplist.readPlist(plist_file_path)

由于服务器配置是存在clowwindy.ShadowsocksX.plist这个文件中的content字段，而content字段通过biplist读出来后是一个字符串类型，格式是json格式的，所以我们可以把它用json库解析成我们方便用的字典  

    import json
    json_config = json.loads(plist["config"])

然后就是将需要更新的内容写进去了。具体写啥就不赘述了。只提下踩到的坑点：
* clowwindy.ShadowsocksX.plist文件中"content"的字段，在Finder下查看是一段经过base64后的字符串，但这并不需要作者手动去base64下那个字段，只要是写入时指定存储进去的内容是Data类型的，plist库就会对应的将其base64后写进去。如果自己手动base64后再当成字符串存储进去，产生的文件是不一样的。使用biplist库则是使用biplist.Data包裹一下要写的内容

        plist["config"] = biplist.Data(json.dumps(json_config, indent=None, separators=(',', ':')))

## 3.2 更新Mac OS X系统中的plist缓存
修改完clowwindy.ShadowsocksX.plist这个文件后，神奇的发现它并没有生效，让我一度怀疑ShadowsocksX的配置文件是不是这个。直接在ShadowsocksX更改配置信息，内容确实会更新到这个文件上面来。然后直接修改这个文件的内容，信息却不会及时更新到ShadowsocksX的界面上去。   
经过n多试验后，发现只有完全重启了mac，才能将修改的配置文件生效。但这显然不够满足我的需求，更新了帐号还要重启那就太蛋疼了。  
谷歌搜索如何立即生效plist文件，能找到的信息实在是少。几经周折，终于在[http://hints.macworld.com/article.php?story=20130908042828630]()这个网页中，在网友的评论里看到相关的信息。
问题所在：   
*  There are many hints here and on the net involving changing user defaults by **running defaults write or directly editing the .plist files in Library/Preferences**. Until 10.9, restarting the program was enough to apply the new defaults.**Since OS X Mavericks**, the defaults system is caching the preferences **system-wide (i.e. not in the application's process!)** to improve performance of the user defaults API. （也就是说，从OS X Mavericks之后，使用mac命令行defaults更改Library/Preferences下面的plist文件后并不会立即生效，必须重启后才生效，因为OS X Mavericks会缓存这些配置，以提高使用defaults读写plist文件时的效率。但是在此却给了我好大烦恼的说～）
* The API documentation states that the cache is synchronized with the on-disk plist file contents periodically, but does not indicate how often, let alone how to flush the cache manually.Logging out and back in appears to flush the user defauts cache, but other than that, *the defaults command is currently the only way to reliably change preferences **without waiting for the timeout***.（说是API文档虽然指出会周期性的同步缓存，但没说是周期是多久。但至少我们知道了defaults操作一下plist文件是可以直接更新缓存的，不用等待系统某个时机去更新缓存）   

看到了问题所在，同时也在评论中看到了解决方法：
* This had me stumped for a while while trying to restore my preference files. So, after copying or editing a plist, for example com.rstudio.desktop.plist , just run **defaults read com.rstudio.desktop** which should *sync the cache*.（也就是说执行一下defaults read **.plist文件就可以更新系统中的plist缓存了）   

那这样我们就可以把更新缓存的代码在python中补上,defaults read一下配置文件。

    os.system("defaults read " + plist_file_path)

# 四、Alfred中按全局快捷键更新服务器信息
用Alfred配置快捷键执行某个脚本挺简单的，也不赘述了。只说下按快捷键后要执行的脚本内容。
1. 为了更新ShadowsocksX的配置，需要把运行中的ShadowsocksX给关掉（当然没有运行的就不用了）。
这个与系统直接打交道的用shell命令简单些。
mac下杀死指定进程的shell命令是kill，不过这个只能通过指定进程的pid来杀，没法通过进程名来杀。  
然后查到有个killall命令，可以杀掉指定进程名的进程（据说这个在unix系统上不能随便执行～）。
不过对于我们来说够用了。执行shell命令:

        killall ShadowsocksX

2. 执行我们刚刚写好的python脚本。    
这边有坑，在Alfred执行的python环境不一定是我们写python脚本时的环境，所以如果无法执行，则只好手动指定python的路径来执行我们的脚本了，例如：

        /Library/Frameworks/Python.framework/Versions/2.7/bin/python ShadowsocksXAutoSetter.py

---
[ShadowsocksXDownload]: https://sourceforge.net/projects/shadowsocksgui/
[ShadowsocksXHelp]: https://github.com/shadowsocks/shadowsocks-iOS/wiki/Shadowsocks-for-OSX-%E5%B8%AE%E5%8A%A9
[iShadowsocks]: http://www.ishadowsocks.net/

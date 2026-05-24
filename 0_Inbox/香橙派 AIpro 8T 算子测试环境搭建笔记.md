## 起因
拿到香橙派 AIpro 8T（昇腾310）后，想验证自己写的自定义算子能否在板子上正确跑通。官方文档偏重推理整体流程，对单算子测试的环境细节写得比较零散。为了后续开发不踩坑，决定从头整理一套可复现的算子测试环境搭建步骤，顺便记录下中间遇到的坑。

## 过程
### 步骤1：烧录镜像

### 步骤2：连接开发版
烧录完镜像之后，网线连接开发板，可以看到网口亮黄灯，且网络适配器中出现以太网即可。记录一下网络配置时容易出现一些坑，以及如何让开发板和电脑连接到一起是一个比较重要的问题。这个文档针对的是minimal镜像，没有图形化界面，应该也兼容[官方用户手册](http://www.orangepi.cn/html/hardWare/computerAndMicrocontrollers/service-and-support/Orange-Pi-AIpro.html)desktop的教程，因此需要按以下步骤进行网络配置：
![image.png](https://picgo.natdog.ccwu.cc/2026/05/03743d1cd4ffab46fabcb3b4136b1005.png)
![image.png|98](https://picgo.natdog.ccwu.cc/2026/05/c87475929da550f5b27eb27304b494a0.png)
### 1. 网络共享
先在cmd终端中查询以太网分配的ip地址，看是否是192.168.137.1,是的话可以直接看第2步，因为ip已经配好了，查询方法是执行 在Windows的终端中执行
```cmd
ipconfig
```
显示如下：
![image.png](https://picgo.natdog.ccwu.cc/2026/05/e13a9bd9bdf132c979c8184496c2fc1f.png)
这里的ipv4地址不是`192.168.137.1`，所以我们需要打开网络共享，将windows电脑当做nat转发机器把网络共享给开发板。
补充：`192.168.137.1`这个是网络共享时设置的本机适配器网络的固定地址，可以把自己电脑看作一个nat转发机器，板子ip就被固定在这个网段下面，可以通过arp -a找到了，这个地址是windows固定设置的，不通过这种方式的话，板子的ip会非常随机，很难扫描到。
![image.png](https://picgo.natdog.ccwu.cc/2026/05/cb3c4fc185361cb503c9f298ee571634.png)
![image.png](https://picgo.natdog.ccwu.cc/2026/05/5318fc9cf9cb9d7069d36d1d9b318ca1.png)
打开网络适配器，找到自己的wifi，编辑更多适配器选项，调整成上图设置，如果一开始就是上图设置，取消`允许其他网络用户...`的勾选，确定之后在重新勾选。
![image.png](https://picgo.natdog.ccwu.cc/{year}/{month}/{md5}.{extName}/20260513153037677.png)
一般来说，此时的以太网ip都会变为`192.168.137.1`
如果还是不变,在powershell里面输入下面命令查看192.168.137.1被谁占用了，谁占用了就准备关谁，把这个ip让出来就好了。
```powershell
Get-NetIPAddress -AddressFamily IPv4 -IPAddress 192.168.137.1
```
### 2. 查询IP
IP分配完成后，在命令行中执行 `arp -a` 命令，查询开发板的IP地址。查询到一个跟192.168.137.1在同一网段的ip
![image.png](https://picgo.natdog.ccwu.cc/{year}/{month}/{md5}.{extName}/20260513153620787.png)
<div align="center"> 
<img src="https://picgo.natdog.ccwu.cc/{year}/{month}/{md5}.{extName}/20260513153620787.png" alt="image.png">                                             


### 3. 登录
获取到开发板的IP地址后，ssh链接它，用户名HwHiAiUser，密码Mind@123。
![image.png](https://picgo.natdog.ccwu.cc/{year}/{month}/{md5}.{extName}/20260513153857890.png)
## 步骤3：昇腾开发环境配置
访问[CANN快速安装地址|160](https://www.hiascend.com/cann/download?version=680&filter=)以安装8.5.0版本为例子，选择对应产品和环境，配置好环境。
![image.png](https://picgo.natdog.ccwu.cc/{year}/{month}/{md5}.{extName}/20260513155729398.png)
![image.png](https://picgo.natdog.ccwu.cc/{year}/{month}/{md5}.{extName}/20260513161453219.png)

@[toc]
## 一、部署类(针对linux服务器)
### 1. 怎么搭建pxe安装环境？(有一个独立的测试网络)
1. 安装所需软件包：`yum install dhcp xinetd syslinux httpd tftp-server -y`
2. 启用tftp，编辑DHCP设置
```


ddns-update-style none;
ignore client-updates;
default-lease-time 259200;
max-lease-time 518400;
option domain-name-servers 10.39.100.150;


subnet 10.39.100.0 netmask 255.255.254.0 {
       range 10.39.100.180 192.168.100.199;
       option routers 10.39.100.150;
       option subnet-mask 255.255.254.0;
       next-server 10.39.100.150;
       # the configuration  file for pxe boot
       filename "pxelinux.0";
}
```
3. 设置镜像文件，配置文件

```
mkdir /var/www/html/centos7
vi /etc/fstab
#加入以下内容
/root/downloads/CentOS-7-x86_64-DVD-1804.iso /var/www/html/centos7/ iso9660 defaults,ro,loop 0 0
```
4. 执行以下命令

```
mount -a
mkdir /var/lib/tftpboot/centos7
cp /var/www/html/centos7/images/pxeboot/initrd.img /var/lib/tftpboot/centos7/
cp /var/www/html/centos7/images/pxeboot/vmlinuz /var/lib/tftpboot/centos7/
cp /usr/share/syslinux/menu.c32 /var/lib/tftpboot
cp /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot
mkdir /var/lib/tftpboot/pxelinux.cfg

```
5. 编辑/var/lib/tftpboot/pxelinux.cfg/default文件，加入以下内容

```
default menu.c32
prompt 0
timeout 300
ONTIMEOUT local

menu title ########## PXE Boot Menu ##########

label 1
menu label ^1) Install CentOS 7 x64 with HTTP
kernel centos7/vmlinuz
append initrd=centos7/initrd.img method=http://10.39.100.150/centos7 devfs=nomount
```

### 2. 怎么使用U盘进行环境安装？
安装前准备：下载系统镜像到U盘，此处以安装Centos为例

1.制作U盘启动盘，此处用UltraISO制作U盘启动盘，选择下载好的Centos镜像
2. 进入BIOS设置启动项优先级，U盘启动优先级最高
3. 保存重新启动系统
4. 重启后进入系统安装界面
5. 设置时区和系统语言
6. 磁盘分区，此处可选用预设模式进行自动分区，也可以自定义分割磁盘。如自定义分区，需要设置挂载点，分区大小，文件类型为EXT3，另外设置swap空间时最好为物理内存的1.5-2倍
7. 设置root密码，创建新用户
8. 进入/etc/sysconfig/network-scripts目录编辑ifcfg-eth0，进行设置IP，DNS，子网掩码等参数，启用启动配置

```

DEVICE="eth0"
TYPE="Ethernet"
ONBOOT="yes"
NM_CONTROLLED="yes"
BOOTPROTO="none"
IPADDR=192.168.217.166
NETMASK=255.255.255.0
GATEWAY=192.168.217.1
DNS1=202.101.172.35

```
9. 重启网络服务：`service network restart`
10. 开启ssh服务
vi /etc/ssh/sshd-config

将以下注释的解开

```
#端口:
port 22

#监听地址：

ListenAddress 0.0.0.0 

ListenAddress : :

#禁止空密码登录:
 PermitEmptyPasswords no

#开启密码登录授权:
 PasswordAuthentication yes


```

 

### 3. 如何进行硬raid的配置和清除？
**硬raid配置**
1. 在启动画面中点击F2键，进入BIOS。

2. 导航到包含SATA Operation mode的类别。

3. 选择SATA Operation Mode。

4. 将模式更改为RAID On然后应用。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200419231151281.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg3NzYwNQ==,size_16,color_FFFFFF,t_70)
5. 保存退出重启系统
6. 启动时按F12进入device configuration
7. 在Intel RAID菜单中选择Create Raid
8. 创建的RAID级别
9. 选择卷和磁条大小，并选择Create Volume
10. 在“创建新阵列”屏幕中，使用箭头键选择 IM 卷的主磁盘（保存有要镜像处理的数据的磁盘）。
11. 使用箭头键移动到此磁盘的“RAID 磁盘”列，然后按 Space 选择是作为值。
12 按 M 进行移植，或按 D 删除该驱动器上的数据。
13.“阵列磁盘”列中的值变为“主”。
14.使用箭头键为 IM 卷选择从（镜像）磁盘，然后选择是作为“阵列磁盘”列的值。

**硬raid清除** 
14. 在 BIOS 启动过程中， 启动 Corp. Configuration Utility。
15. 在基于 BIOS 的配置实用程序的主菜单上使用箭头键选择一个适配器。 enter 转入“适配器属性”屏幕。
16. 在“选择新阵列类型”屏幕中，使用箭头键选择新建现有阵列。
17. 在“查看阵列”屏幕中，使用箭头键选择管理阵列。 
18. 在“管理阵列”屏幕中，使用箭头键选择删除阵列。 

### 4. linux上如何配置nfs服务器，以及客户端上进行挂载？
一、 nfs服务器配置
1. 安装nfs服务端

```
 yum -y install nfs-utils
```
2. 创建共享目录并设置权限

```
mkdir /data/nfs179
    chown -R nfsnobody.nfsnobody /data/nfs179
```
3. 配置共享目录

```shell
vim /etc/exports
#增加如下内容
/data/nfs179    *(rw,sync,no_root_squash,no_subtree_check,insecure)
```
4. 启动服务并开启防火墙端口

```
systemctl start nfs
firewall-cmd --add-service=nfs --permanent --zone=public
firewall-cmd --add-service=mountd --permanent --zone=public
firewall-cmd --add-service=rpc-bind --permanent --zone=public
firewall-cmd --reload    
```

二、 客户端挂载
1. 客户端安装

```
yum -y intall nfs-utils
```
2. 挂载到本地并测试读写

```
mkdir /mnt/nfs
mount -t nfs nfs_ip:mount_dir /mnt/nfs
touch /mnt/nfs/111.txt
echo "2222" >> 111.txt
```

### 5. 已经装好的环境如何进行磁盘扩容？
1. 创建新分区
```shell
fdisk  /dev/hda

n

l        #选择逻辑分区，如果没有，则首先创建扩展分区，然后再添加逻辑分区（硬盘：最多四个分区P-P-P-P或P-P-P-E）

6        #分区号（从5开始），/dev/hda6

t      8e   #分区类型8e表示LVM分区

w        #写入分区表

partprobe   #重读分区表

mkfs –t ext3 /dev/hda6 #格式化

partx /dev/hda #查看当前硬盘的分区表及使用情况
```
2. 创建PV，扩容VG，LV

```shell
pvcreate /dev/hda6

vgdisplay #查看当前已经存在的VG信息，以存在VG：VolGroup00为例

vgextend VolGroup00 /dev/hda6    #扩展VolGroup00

lvdisplay #查看已经存在的LV信息，以存在LV：LogVol01为例

lvextend –L 1G /dev/VolGroup00/LogVol01 #扩展LV

resize2fs /dev/VolGroup00/LogVol01 #执行该重设大小，对于当前正在使用的LogVol01有效

```
## 二、脚本实现
远程调用实现EXSI服务器中的虚拟机：安装、还原、关机、开启、添加网卡、断开网络连接
```shell
#!/bin/bash
# encoding: utf-8


#脚本使用命令帮助
usage() {
    echo "Usage: sh $0 [ create|restore|shutdown|start|addNetcard|delNetcard]"
    exit 1
}

#检测虚机是否已在运行状态
isExist(){
	state=`esxcli vm process list| grep $1 | awk '{print $2}'`
	if [ "$state" != "" ];then
		echo ">> VM is already running"
		resExist=1
	else
		echo ">> VM begin to start"
		resExist=0
	fi
}

#恢复虚机
restore(){
	nbrestorevm - vmw - C $1 - O
	echo "Restore VM is successful!"
}

#安装虚机
create(){
	New-VM -Name $1 -VMHost $hostInCluster1 -ResourcePool ( Get-ResourcePool DevelopmentResources ) -DiskMB 4000 -MemoryMB 256
	echo "$1 is created!"
}
# 添加网卡
addNetcard(){
	
	esxcli network vswitch standard uplink add --uplink-name=vmnic --vswitch-name=vSwitch
	echo "Adding network card is successful!"
}

# 删除网卡
delNetcard(){
	esxcli network vswitch standard uplink remove --uplink-name=vmnic --vswitch-name=vSwitch # unlink an uplink
	echo "Deleting network card is successful!"
}

#关机
shutdown(){
	/usr/lib/vmware-vcli/apps/host/hostops.pl --target_host ESX-Host-FQDN --operation shutdown --url https://vCenter-Host/sdk/vimService.wsdl
	echo "Shutdown is successful!"
}
#启动
start(){
	isExist
	if [ "$resExist" == 1 ];then
		echo ">> Fail to start VM"
	else
		vim-cmd vmsvc/power.on 
		echo "Starting VM is successful!"
	fi
	
}


main(){
	case "$1" in
		"addNetcard")
			addNetcard
			;;
		"delNetcard")
			delNetcard
			;;
		"shutdown")
			shutdown
			;;
		"start")
			start $2
		;;
		"restore")
			restore $2
		;;
		"create")
			create $2
		;;
		*)
			usage
		;;
	esac

}


main $1 $2


```
## 三、高可用和负载均衡的测试从哪些方面进行考虑？
1.健康检查测试
常用ICMP健康检查，关注服务器有效时的状态，关注服务宕机，设备能否及时感知并切换状态。
服务器故障处理测试时，故障发生，观察健康检查状态，记录发现服务器宕机时间和将所有用户切到另一台服务器的时间；故障恢复，观察健康检查状态，查看服务器恢复时间，客户恢复访问该实服务的时间
2.会话保持测试
  在某些情况下，一次业务交互可能包括多个连接，多个连接之间可能存在隐含的关联，需要将多个连接持续重定向到同一个服务器，通过创建会话保持表项来实现持续性功能，保证请求的正确响应。
  3.高可靠测试
  关注大量客户访问情况下两台负载均衡服务器之间，会话、持续性表项备份的正确性、完备性、实时性，保证主设备故障后，备设备接着工作，业务不中断。
  4.虚拟负载均衡测试
  在规格范围内所有虚负载均衡同时工作的情况下整个系统的稳定性。关注每个虚拟SLB的cpu、内存、资源的消耗情况。
## 四、 针对以下API进行用例设计
一般针对用例设计分为四大块：
- 正向测试：填入符合api文档的参数调用接口，检查返回值是否符合预期，涉及post，delete，put修改数据库的，需要调用查询接口或者去数据库验证是否接口调用生效，最后需调用清除本次测试数据，以保证下次测试的正常执行
以验证创建主机资产该接口中的**sysType**为例设计测试用例：
针对sysType中的9个不同参数对应的不同的设备厂商设计9条用例,下图仅列出当sysType为1时的用例
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200421002025901.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg3NzYwNQ==,size_16,color_FFFFFF,t_70)

- 逆向测试：逆向测试包括参数类型异常，参数缺失，为空，参数超出范围，重复提交
此处以type参数为例，设计逆向用例
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200421003842249.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg3NzYwNQ==,size_16,color_FFFFFF,t_70)
- 性能测试：
1. 在jmter上建立线程组，设置线程数和循环次数分别1000，100
2. 在线程组中添加http请求元器件，配置ip,port,body数据
3. 添加响应断言，判断响应是否为201
4. 在接口服务器上安装nmon，监测服务器实时cpu，内存，io等使用i情况
5. 执行`jmeter -n -t testplan/RedisLock.jmx -l testplan/result/result.txt -e -o testplan/webreport`
6. 导出nmon数据，利用nmon_analyser，生成监控数据报告图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200421004919609.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg3NzYwNQ==,size_16,color_FFFFFF,t_70)
### 使用python实现以下API的调用
先将数据文件写入excel中，实现数据，代码分离，也方便以后的管理。
```python
import xlrd
import json
import requests
 
path='datafile path'
#打开的Excls
data=xlrd.open_workbook(path)
nums=len(data.sheets())
#返回第一个sheet
sheet1=data.sheet_by_index(0)
url=sheet1.cell(1,3).value
path=sheet1.cell(1,4).value
url="http://"+url+path
method=sheet1.cell(1,5).value
header=sheet1.cell(1,6).value
body=sheet1.cell(1,7).value
 
print(type(header))
#转换成dict格式
header=json.loads(header)
#转换成dict格式
body=json.loads(body)
print(header,type(header))
print(body,type(body))
 
 
response=requests.post(url=url,headers=header,json=body)
print(response.text)
```

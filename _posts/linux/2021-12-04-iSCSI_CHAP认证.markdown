# 登录CHAP认证的iSCSI target

使用iStorage Server为linux服务器创建iSCSI target ，iStorage Server 支持CHAP在内的多种认证方法，而CHAP是目前IP SAN领域最常用的安全机制。

iStorage Server是kernsafe旗下的一款优秀的IP SAN服务器软件，创建target方法，可以查看官网白皮书：www.kernsafe.cn



#### 第一种：全局配置的方法，客户端上连接的所有target可能都会使用这个账号和密码进行认证，适用于只有一个target



```
[root@www ~]# yum -y install iscsi-initiator-utils
[root@www ~]# vi /etc/iscsi/iscsid.conf
# line 49: uncomment
node.session.auth.authmethod = CHAP
# line 53,54: uncomment and set username and password which set on iSCSI Target
node.session.auth.username = username
node.session.auth.password = password
# discover target
[root@www ~]# iscsiadm -m discovery -t sendtargets -p 10.0.0.30
Starting iscsid: Loading iSCSI transport class v2.0-870.
iscsi: registered transport (tcp)
iscsi: registered transport (iser)
cxgb3i: tag itt 0x1fff, 13 bits, age 0xf, 4 bits.
iscsi: registered transport (cxgb3i)
cnic: Broadcom NetXtreme II CNIC Driver cnic v2.1.2 (May 26, 2010)
Broadcom NetXtreme II iSCSI Driver bnx2i v2.1.1 (Mar 24, 2010)
iscsi: registered transport (bnx2i)
iscsi: registered transport (be2iscsi)
[ OK ]
10.0.0.30:3260,1 iqn.2011-07.world.server:target0
[root@www ~]# chkconfig iscsi on
[root@www ~]# chkconfig iscsid on
# login to target
[root@www ~]# iscsiadm -m node --login
Logging in to [iface: default, target: iqn.2011-07.world.server:target0, portal: 10.0.0.30,3260]
scsi2 : iSCSI Initiator over TCP/IP
scsi 2:0:0:0: RAID IET Controller 0001 PQ: 0 ANSI: 5
scsi 2:0:0:1: Direct-Access IET VIRTUAL-DISK 0001 PQ: 0 ANSI: 5
Login to [iface: default, target: iqn.2011-07.world.server:target0, portal: 10.0.0.30,3260] successful.
# confirm session
[root@www ~]# iscsiadm -m session -o show
tcp: [1] 10.0.0.30:3260,1 iqn.2011-07.world.server:target0
# confirm partitions
[root@www ~]# cat /proc/partitions
major minor #blocks name
8031457280sda
81512000sda1
8230944256sda2
253020971520dm-0
25316160384dm-1
25323809280dm-2
80104857600sdb
81104857600sdb1# added new device provided from target
# config for auto-mount when booting
[root@www ~]# vi /etc/fstab
/dev/mapper/VolGroup-lv_root / ext4 defaults 1 1
UUID=3d3f19a1-582f-4a29-a304-349750094b2c /boot ext4 defaults 1 2
/dev/mapper/VolGroup-lv_swap swap swap defaults 0 0
tmpfs /dev/shm tmpfs defaults 0 0
devpts /dev/pts devpts gid=5,mode=620 0 0
sysfs /sys sysfs defaults 0 0
proc /proc proc defaults 0 0
# add iSCSI filesystem
/dev/sdb1 /var/kvm ext4 _netdev 0 0

[root@www ~]# chkconfig netfs on
________________________________________________________________________________________________
```
#### 方法二：对特定的target进行修改(假如我们同时连接了多个target，必须这样)
```
http://zlbzhu.blog.51cto.com/1413424/897422
发现target
iscsiadm -m discovery -t sendtargets -p 192.168.1.211
挂载target
iscsiadm -m node -T iqn_2013-06.com.safexjt:target00 -p 192.168.1.211 -l
配置该target,默认放在客户端/var/lib/iscsi/node下面
vi /var/lib/iscsi/nodes/iqn_2013-06.com.safexjt\:target00/192.168.1.211\,3260\,1/default
在node.session.*.* 段添加如下信息
node.session.auth.authmethod = CHAP
node.session.auth.username = jack
node.session.auth.password = jack
保存
#service iscsi restart
________________________________________________________________________________________________
```
#### 方法三：如果我们已经对target进行了discovery，但是在挂载的时候提示认证失败，这个时候我们也可以通过iscsiadm明明进行添加认证信息
```
root@iscsi /]#iscsiadm -m node -T iqn_2013-06.com.safexjt:target00 -p 192.168.1.211:3260 -o update --name=node.session.auth.authmethod --value=CHAP
[root@iscsi /]#iscsiadm -m node -T iqn_2013-06.com.safexjt:target00 -p 192.168.1.211:3260 -o update --name= node.session.auth.username --value=xxxxxxx
[root@iscsi /]#iscsiadm -m node -T iqn_2013-06.com.safexjt:target00 -p 192.168.1.211:3260 -o update --name= node.session.auth.password --value=xxxxxxx
需要注意的是，发现Target的命令（iscsiadm -m discovery -t sendtargets）会自动按照/etc/iscsi/iscsi.conf文件中的参数配置刷新/var/lib/iscsi/nodes下initiator登录target要使用的参数文件，所以如果通过修改/var/lib/iscsi/nodes下的文件设置好CHAP认证后又对该存储服务器执行了发现target的操作，则需要再次修改该文件。
```









---

----



### 一、单向认证 

单向认证是仅initiator连接target时进行认证。这个配置比较简单，打开/etc/iscsi/iscsid.conf 文件，找到如下三项并取消注释：



```bsh
node.session.auth.authmethod = CHAP   //开启CHAP认证
node.session.auth.username = redhat    //配置账号
node.session.auth.password = redhat123  //密码
```

通过以下命令重启服务并生效：

```bsh
/etc/init.d/iscsid restart
或
systemctl restart iscsid
```

上面的配置只是在开机自启动时会查找该配置中的信息，如果想要通过命令行测试用户名密码的有效性可以使用如下命令：



```bsh
# 发现target
iscsiadm -m discovery -t sendtargets -p 10.131.131.150  //发现
iscsiadm -m node -T iqn.2006-08.com.huawei:oceanstor:10.131.131.150 -l   //登陆
iscsiadm -m node -o delete -T iqn.2006-08.com.huawei:oceanstor:10.131.131.150 //删除session
# 使用用户名密码
iscsiadm -m node -o update -p 10.131.131.150 -n node.session.auth.authmethod -v CHAP
iscsiadm -m node -o update -p 10.131.131.150 -n node.session.auth.username -v myusername
iscsiadm -m node -o update -p 10.131.131.150 -n node.session.auth.password -v mypassword
# 也可以使用下面的格式
iscsiadm -m node -T iqn.2006-08.com.huawei:oceanstor:10.131.131.150 -o update --name node.session.auth.authmethod --value=CHAP
iscsiadm -m node -T iqn.2006-08.com.huawei:oceanstor:10.131.131.150 --op update --name node.session.auth.username --value=myusername
iscsiadm -m node -T iqn.2006-08.com.huawei:oceanstor:10.131.131.150 --op update --name node.session.auth.password --value=mypassword
```

连接后，可以使用如下命令查看连接状态：



```bsh
# 查看状态
iscsiadm -m node
iscsiadm -m node -o show
# 查看连接后状态
iscsiadm -m session -o show
```

上面的连接状态信息，也可以通过进入/var/lib/iscsi/ifaces/目录通过查看文件进行查看。对于已经连接过的session，可以使用如下命令删除该session：



```bsh
for iqn in `iscsiadm -m node|awk '{print $NF}'`;do iscsiadm -m node -T $iqn -u;done
for iqn in `iscsiadm -m node|awk '{print $NF}'`;do iscsiadm -m node -o delete -T $iqn;done
```

### 二、双向认证 

双向认证就是在Initiator端和target端都进行认证。Initiator 认证：在initiator尝试连接到一个target的时候，initator需要提供一个用户名和密码给target供target进行认证。下面我们称这个用户名密码为incoming账号，即：incoming账号是initiator端提供给target端，供target端认证的账号。target 认证：在initiator尝试连接到一个target的时候，target需要提供一个用户名和密码给initiator供initiator进行认证。与之对应的是outcoming账号，即：outcoming账号是target端提供给initiator端，供initiator认证的账号。双向认证的配置如下：



```bsh
# To set a CHAP username and password for initiator
# authentication by the target(s), uncomment the following lines:
node.session.auth.username = username
node.session.auth.password = password
# To set a CHAP username and password for target(s)
# authentication by the initiator, uncomment the following lines:
node.session.auth.username_in = username_in
node.session.auth.password_in = password_in
```

### 三、有关多路径聚合 

存储端一般会有多控，也就是可以由多个对应的IP可供discovery 和 login使用，比如一个两控的就会使用如下的方式进行挂载：

```bsh
# iscsiadm -m node -p <存储系统A控iSCSI主机端口的IP-A> -l# iscsiadm -m node -p <存储系统B控iSCSI主机端口的IP-B> -l
```

接下来可以使用多路径软件进行聚合，这里以multipath为例，使用如下指令生成配置文件并重启服务生效。



```bsh
# mpathconf --enable# mpathconf --with_module y# mpathconf --with_multipathd y
```
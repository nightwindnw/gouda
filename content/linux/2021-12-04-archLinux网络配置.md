+++
title =   "ArchLinux网络配置"
date =   "2021-12-26"
[taxonomies] 
tags = ["linux","archlinux"]
categories = ["linux"]
+++

### Archlinux 网络配置方法

在 Archlinux 下最简单的网络配置方式是使用 `systemd-networkd` 服务，默认配置文件位于 `/etc/systemd/network` 下

使用 `systemctl` 开启 `systemd-networkd` 服务

``` bash
$ systemctl enable systemd-networkd
```

#### 配置 DHCP 动态 IP 地址

编辑 `/etc/systemd/network/enp0s3.network` 网卡配置文件

```
[Match]
Name=enp0s3

[Network]
DHCP=ipv4
```

#### 配置 STATIC 静态 IP 地址

编辑 `/etc/systemd/network/enp0s3.network` 网卡配置文件

```
[Match]
Name=enp0s3

[Network]
Address=192.168.1.100/24
Gateway=192.168.1.1
DNS=8.8.8.8
DNS=8.8.4.4
```

如果 enp0s3.network 中添加了 DNS 选项则需要同时启用 `systemd-resolved.service` 服务配合使用

```
$ systemctl enable systemd-resolved.service
```

#### 重启网络服务，使配置生效

修改网卡配置文件之后，需要重启 `systemd-networkd` 服务使配置生效

```
$ systemctl restart systemd-networkd
```
---
layout: post
title:  "linux下验证启动模式是BIOS还是UEFI"
date:   2021-12-04
categories: linux
---
验证启动模式
如果以在 UEFI 主板上启用 UEFI 模式，Archiso 将会使用 systemd-boot 来 启动 Arch Linux。可以列出 efivars 目录以验证启动模式：

# ls /sys/firmware/efi/efivars
如果目录不存在，系统可能以 BIOS 或 CSM 模式启动，详见您的主板手册。
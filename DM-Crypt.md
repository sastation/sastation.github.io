---
title: DM-Crypt
date: 2016-10-08 15:15
cdn: header-on
header-img: title06.jpg

---

# DM-Crypt

## 常用命令
```bash
    cryptsetup --version
    cryptsetup benchmark
```

## 创建 LUKS 的物理加密盘
```bash
    # 创建用于加密的物理分区
    parted /dev/sdbmkpa
        mklabel [msdos|gpt]
        mkpart
            ext3
            1
            50GB
        print

    # 创建加密分区
    #cryptsetup --cipher aes-xts --key-size 256 --hash sha256 --iter-time 10000 luksFormat /dev/sdb1
    cryptsetup luksFormat /dev/sdb1 # phrase: (Sync02)
    # 打开加密分区
    cryptsetup luksOpen /dev/sdb1 crypt-d1 # phrase: (Sync02)
    cryptsetup status crypt-d1 # view status
    ls /dev/mapper # verify
    # 创建文件系统
    mkfs.ext4 /dev/mapper/crypt-d1
```

## 【挂载|卸载】加密分区
```bash
    cryptsetup luksOpen /dev/sdb1 crypt-d1
    mount /dev/mapper/crypt-d1 /home/sync
    
    umount /home/sync
    cryptsetup luksClose crypt-d1
```

- [dm-crypt 扫盲](https://program-think.blogspot.com/2015/10/dm-crypt-cryptsetup.html)
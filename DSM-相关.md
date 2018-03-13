---
title: DSM 相关
date: 2016-12-06 16:54
tags: 
    - 技术
    - DSM

---

# DSM 相关

## 在VMware中安装DSM 5.2

### 选择 "其他Linux 内核3.x"

### 添加5644引导光盘为启动镜像

### 修改 .vmx 文件，添加如下一行
bios.bootOrder = "cdrom,hdd,floppy"

### 安装
- 5644.pat

### 更新顺序
- 5644-U1
- 5967
- 5967-U2

### 关闭自动更新

### 安装套件
- Cloud Station
- Git

## Megrate Volume in DSM

### 注意事项
- create new volume (basic volume, jbod volume, RAID volume)
- CloudStation 只能在安装时选定的卷名上正常工作（只需要卷名相同）

### 转移数据
- 控制面板>共享文件夹>编辑>常规>所在位置>[选择新的卷]

### 转移套件
**注意：若新旧卷名不一致，移动@CloudStation仍会引起CloudStation重建**

- 停止所有套件
- ssh to dsm
- move @appstore to new volume
- move @CloudStation to new volume
- move @database to new volume
- move all necessary folders to new volume
- link all /var/packages/[appname]/target to new location

### 重新使用原卷名
**注意：DSM会自动顺序编号，若要使用原有卷名，需在编号中空出原卷名**

- 确认所有数据已转移到新卷，所有套件已经停止
- 删除旧的卷
- 确认需要使用的旧卷名已空出，并在第一顺位
- 重建卷
- 转移数据
- 转移套件
- 重启（可选）
- 启动所有停止的套件

### 创建 JBOD 的方法
- 需要添加至少两块硬盘
- 在存储空间中创建 JBOD 卷
    - 存储空间中创建的是软 RAID，只能创建一个卷，能添加新硬盘（命令行：mdadm）
    - 磁盘群组中创建的是 LVM 设备，能在此之上创建多个卷（命令行：lvm）

### 手工转移数据的脚本
```bash
/var/packages/CloudStation/scripts/start-stop-status stop
/var/packages/Git/scripts/start-stop-status stop
/var/packages/NoteStation/scripts/start-stop-status stop

mv /volume1/@appstore /volume2
mv /volume1/@cloudstation /volume2
mv /volume1/@database /volume2

#rm -fv /var/packages/CloudStation/target/
ln -sf /volume2/@appstore/CloudStation /var/packages/CloudStation/target

#rm -fv /var/packages/Git/target/
ln -sf /volume2/@appstore/Git /var/packages/Git/target

#rm -fv /var/packages/NoteStation/target/
ln -sf /volume2/@appstore/NoteStation /var/packages/NoteStation/target

reboot
```

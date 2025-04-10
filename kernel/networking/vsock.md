# Vsock

## 获取 vsock local CID
* vsock 目前只有一个 `ioctl()` command，`IOCTL_VM_SOCKETS_GET_LOCAL_CID`，可以 python 脚本 `vsock-get-cid.py` 获取
```py
#!/usr/bin/env python3
import socket
import struct
import fcntl

with open("/dev/vsock", "rb") as fd:
    r = fcntl.ioctl(fd, socket.IOCTL_VM_SOCKETS_GET_LOCAL_CID, "    ")
    cid = struct.unpack("I", r)[0]

    print("Local CID: {}".format(cid))
```

## 虚拟机内用 vsock 与 Host 通信

* Host 侧需安装以下 modules
```sh
# lsmod | grep -i vsock
Module                  Size  Used by
vhost_vsock            24576  1
vmw_vsock_virtio_transport_common    61440  1 vhost_vsock
vhost                  69632  1 vhost_vsock
vsock                  61440  3 vmw_vsock_virtio_transport_common,vhost_vsock,vsockmon
```

* Geust 侧需安装以下 modules
```sh
# lsmod | grep -i vsock
Module                  Size  Used by
vmw_vsock_virtio_transport    20480  0
vmw_vsock_virtio_transport_common    61440  1 vmw_vsock_virtio_transport
vsock                  65536  2 vmw_vsock_virtio_transport_common,vmw_vsock_virtio_transport
```
* 对应 config 为：
```c
CONFIG_VSOCKETS=y
CONFIG_VIRTIO_VSOCKETS_COMMON=y
CONFIG_VIRTIO_VSOCKETS=y
```

### `CID` 的指定

* Host 侧监听用的 `CID` 可以是 `VMADDR_CID_HOST (2)` 或者 `VMADDR_CID_ANY (-1)`，但不能任意指定。为什么这样，见后面的代码部分
* Guest 侧的 `CID` 通过 Qemu 配置 `-device vhost-vsock-pci,guest-cid=${VSOCK_GUEST_CID}` 来指定

### C 语言版本

#### Host 收包

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <linux/vm_sockets.h>

#define CID  VMADDR_CID_HOST
#define PORT 4050
#define BUFFER_SIZE 1024

int main() {
    int server_fd, client_fd;
    struct sockaddr_vm server_addr, client_addr;
    socklen_t client_addr_len = sizeof(client_addr);
    char buffer[BUFFER_SIZE];
    ssize_t bytes_read;

    // 创建 VSOCK 套接字
    server_fd = socket(AF_VSOCK, SOCK_STREAM, 0);
    if (server_fd < 0) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    // 设置服务器地址
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.svm_family = AF_VSOCK;
    server_addr.svm_cid = CID; // 主机侧 CID
    server_addr.svm_port = PORT;

    // 绑定套接字
    if (bind(server_fd, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("bind");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    // 监听连接
    if (listen(server_fd, 5) < 0) {
        perror("listen");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    printf("VSOCK server is listening on CID=%u, port=%d...\n", CID, PORT);

    // 接受客户端连接
    client_fd = accept(server_fd, (struct sockaddr *)&client_addr, &client_addr_len);
    if (client_fd < 0) {
        perror("accept");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    printf("Client connected: CID=%u, port=%u\n", client_addr.svm_cid, client_addr.svm_port);

    // 接收消息
    while ((bytes_read = read(client_fd, buffer, BUFFER_SIZE))) {
        if (bytes_read < 0) {
            perror("read");
            break;
        }
        buffer[bytes_read] = '\0'; // 确保字符串以 null 结尾
        printf("Received message: %s\n", buffer);
    }

    // 关闭连接
    close(client_fd);
    close(server_fd);
    printf("Server closed.\n");

    return 0;
}
```
* 打印如下：
```sh
# ./vsock-recv
VSOCK server is listening on CID=2, port=4050...
Client connected: CID=4, port=4079680478
Received message: Hello, Host! This is a message from the VM.
Server closed.
```

#### Guest 发包
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <linux/vm_sockets.h>

#define HOST_CID  VMADDR_CID_HOST // 主机侧的 CID
#define PORT 4050  // 主机侧监听的端口
#define BUFFER_SIZE 1024

int main() {
    int sock_fd;
    struct sockaddr_vm server_addr;
    const char *message = "Hello, Host! This is a message from the VM.";
    char buffer[BUFFER_SIZE];
    ssize_t bytes_sent, bytes_received;

    // 创建 VSOCK 套接字
    sock_fd = socket(AF_VSOCK, SOCK_STREAM, 0);
    if (sock_fd < 0) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    // 设置服务器地址（主机侧）
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.svm_family = AF_VSOCK;
    server_addr.svm_cid = HOST_CID; // 主机侧的 CID=5
    server_addr.svm_port = PORT;   // 主机侧监听的端口

    // 连接到主机
    if (connect(sock_fd, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("connect");
        close(sock_fd);
        exit(EXIT_FAILURE);
    }

    printf("Connected to host: CID=%u, port=%u\n", HOST_CID, PORT);

    // 发送消息到主机
    bytes_sent = send(sock_fd, message, strlen(message), 0);
    if (bytes_sent < 0) {
        perror("send");
        close(sock_fd);
        exit(EXIT_FAILURE);
    }
    printf("Sent message to host: %s\n", message);
#if 0
    // 接收主机的回复（可选）
    bytes_received = recv(sock_fd, buffer, BUFFER_SIZE, 0);
    if (bytes_received < 0) {
        perror("recv");
        close(sock_fd);
        exit(EXIT_FAILURE);
    }
    buffer[bytes_received] = '\0'; // 确保字符串以 null 结尾
    printf("Received reply from host: %s\n", buffer);
#endif
    // 关闭连接
    close(sock_fd);
    printf("Connection closed.\n");

    return 0;
}
```
* 打印如下：
```sh
# ./vsock-snd
Connected to host: CID=2, port=4050
Sent message to host: Hello, Host! This is a message from the VM.
Connection closed.
```

### Python 版本

* Host 侧
```py
#!/usr/bin/env python3

import socket

CID = socket.VMADDR_CID_HOST
PORT = 4050

s = socket.socket(socket.AF_VSOCK, socket.SOCK_STREAM)
s.bind((CID, PORT))
s.listen()
(conn, (remote_cid, remote_port)) = s.accept()

print(f"Connection opened by cid={remote_cid} port={remote_port}")

while True:
    buf = conn.recv(64)
    if not buf:
        break

    print(f"Received bytes: {buf}")
```

* Guest 侧
```py
#!/usr/bin/env python3

import socket

CID = socket.VMADDR_CID_HOST
PORT = 4050

s = socket.socket(socket.AF_VSOCK, socket.SOCK_STREAM)
s.connect((CID, PORT))
s.sendall(b"Hello, world!")
s.close()
```

## Host 侧抓 vsock 包
* Host 侧可监控 vsock 的 traffic 如下：
```sh
modprobe vsockmon
ip link add type vsockmon
ip link set vsockmon0 up
tcpdump -n -i vsockmon0
wireshark -k -i vsockmon0
```
* 可抓到 packets 如下：
```c
03:30:52.706188 VIRTIO 4.4079680474 > 2.4050 CONNECT, length 76
03:30:52.706219 VIRTIO 2.4050 > 4.4079680474 CONNECT, length 76
03:30:52.707703 VIRTIO 4.4079680474 > 2.4050 PAYLOAD, length 119
03:31:08.695536 VIRTIO 4.4079680474 > 2.4050 DISCONNECT, length 76
03:31:08.695548 VIRTIO 2.4050 > 4.4079680474 DISCONNECT, length 76
03:31:08.696540 VIRTIO 2.4050 > 4.4079680474 DISCONNECT, length 76
```
* 解释如下，
  * Host 监听在 `CID = VMADDR_CID_HOST` 端口为 `4050`
  * Guest 由于 Qemu 配置了 `-device vhost-vsock-pci,guest-cid=4`，因此发包 `CID` 是 `4`，端口未指定，是随机值

## Host 绑定 vsock
* Host 用的是 `vhost-vsock.ko`，初始化时把 `vhost_transport.transport = transport_h2g`
```c
//net/vmw_vsock/af_vsock.c
vhost_vsock_init()
-> vsock_core_register(&vhost_transport.transport, VSOCK_TRANSPORT_F_H2G)
-> misc_register(&vhost_vsock_misc) //注册 /dev/vhost-vsock
```
* Host 绑定 vsock 时走到以下路径
```c
//net/vmw_vsock/af_vsock.c
vsock_bind()
-> __vsock_bind(sk, vm_addr)
```
* net/vmw_vsock/af_vsock.c
```c
//在绑定 socket 表中查找给定的 vsock 地址，返回 socket
static struct sock *__vsock_find_bound_socket(struct sockaddr_vm *addr)
{
    struct vsock_sock *vsk;
    //遍历绑定 socket 表
    list_for_each_entry(vsk, vsock_bound_sockets(addr), bound_table) {
        if (vsock_addr_equals_addr(addr, &vsk->local_addr))
            return sk_vsock(vsk); //如果找到 CID 和 port 完全相等的 socket，返回它
        //如果找到 port 相等，且 给定的 socket 的 CID 或 表项中的 CID 为 VMADDR_CID_ANY 时，也认为找到了，返回它
        if (addr->svm_port == vsk->local_addr.svm_port &&
            (vsk->local_addr.svm_cid == VMADDR_CID_ANY ||
             addr->svm_cid == VMADDR_CID_ANY))
            return sk_vsock(vsk);
    }
    //找不到，返回 NULL
    return NULL;
}

bool vsock_find_cid(unsigned int cid)
{   //Qemu 参数 -device vhost-vsock-pci,guest-cid=${VSOCK_GUEST_CID}" 给的 CID？？Guest 会走进这个条件
    if (transport_g2h && cid == transport_g2h->get_local_cid())
        return true;
    //Host CID 如果不用 ANY 只能用 CID = VMADDR_CID_HOST = 2，否则认为找不到 CID，并最终让 bind() 返回 -EADDRNOTAVAIL！！
    if (transport_h2g && cid == VMADDR_CID_HOST)
        return true;

    if (transport_local && cid == VMADDR_CID_LOCAL)
        return true;

    return false;
}

static int __vsock_bind_connectible(struct vsock_sock *vsk,
                    struct sockaddr_vm *addr)
{
    static u32 port;
    struct sockaddr_vm new_addr;

    if (!port)
        port = get_random_u32_above(LAST_RESERVED_PORT);

    vsock_addr_init(&new_addr, addr->svm_cid, addr->svm_port);
    //Host port 传入为 VMADDR_PORT_ANY
    if (addr->svm_port == VMADDR_PORT_ANY) {
        bool found = false;
        unsigned int i;

        for (i = 0; i < MAX_PORT_RETRIES; i++) {
            if (port <= LAST_RESERVED_PORT)
                port = LAST_RESERVED_PORT + 1;

            new_addr.svm_port = port++;

            if (!__vsock_find_bound_socket(&new_addr)) {
                found = true;
                break;
            }
        }

        if (!found)
            return -EADDRNOTAVAIL;
    } else {
        /* If port is in reserved range, ensure caller
         * has necessary privileges.
         */
        if (addr->svm_port <= LAST_RESERVED_PORT &&
            !capable(CAP_NET_BIND_SERVICE)) {
            return -EACCES;
        }
        //如果找到，返回正在使用中的错误，否则往下走，socket 加入绑定 socket 表
        if (__vsock_find_bound_socket(&new_addr))
            return -EADDRINUSE;
    }

    vsock_addr_init(&vsk->local_addr, new_addr.svm_cid, new_addr.svm_port);

    /* Remove connection oriented sockets from the unbound list and add them
     * to the hash table for easy lookup by its address.  The unbound list
     * is simply an extra entry at the end of the hash table, a trick used
     * by AF_UNIX.
     */
    __vsock_remove_bound(vsk);
    __vsock_insert_bound(vsock_bound_sockets(&vsk->local_addr), vsk); //插入绑定 socket 表

    return 0;
}

static int __vsock_bind(struct sock *sk, struct sockaddr_vm *addr)
{
    struct vsock_sock *vsk = vsock_sk(sk);
    int retval;

    /* First ensure this socket isn't already bound. */
    if (vsock_addr_bound(&vsk->local_addr))
        return -EINVAL;
    //Host 绑定的 CID 不能任意指定，只能是 VMADDR_CID_ANY 或者 VMADDR_CID_HOST
    /* Now bind to the provided address or select appropriate values if
     * none are provided (VMADDR_CID_ANY and VMADDR_PORT_ANY).  Note that
     * like AF_INET prevents binding to a non-local IP address (in most
     * cases), we only allow binding to a local CID.
     */
    if (addr->svm_cid != VMADDR_CID_ANY && !vsock_find_cid(addr->svm_cid))
        return -EADDRNOTAVAIL;

    switch (sk->sk_socket->type) {
    case SOCK_STREAM:
    case SOCK_SEQPACKET:
        spin_lock_bh(&vsock_table_lock);
        retval = __vsock_bind_connectible(vsk, addr);
        spin_unlock_bh(&vsock_table_lock);
        break;

    case SOCK_DGRAM:
        retval = __vsock_bind_dgram(vsk, addr);
        break;

    default:
        retval = -EINVAL;
        break;
    }

    return retval;
}
```

## Guest 连接 Host 流程
* Qemu 使用的是 `vmw_vsock_virtio_transport.ko`，初始化时把 `virtio_transport.transport = transport_g2h`
```c
//net/vmw_vsock/virtio_transport.c
virtio_vsock_init()
-> virtio_vsock_workqueue = alloc_workqueue("virtio_vsock", 0, 0)
-> vsock_core_register(&virtio_transport.transport, VSOCK_TRANSPORT_F_G2H) //这里会把 virtio_transport.transport = transport_g2h
-> register_virtio_driver(&virtio_vsock_driver)
```
* Guest 连接的时候路径如下
```c
//net/vmw_vsock/af_vsock.c
vsock_connect()
   switch (sock->state)
   default:
-> memcpy(&vsk->remote_addr, remote_addr, sizeof(vsk->remote_addr))
-> vsock_assign_transport(vsk, NULL)
-> vsock_auto_bind(vsk)
   sk->sk_state = TCP_SYN_SENT;
-> transport->connect(vsk)
   //net/vmw_vsock/virtio_transport_common.c
=> virtio_transport_connect()
   -> virtio_transport_send_pkt_info(vsk, &info)
      -> t_ops->send_pkt(skb)
         //net/vmw_vsock/virtio_transport.c
      => virtio_transport_send_pkt()
         -> virtio_vsock_skb_queue_tail(&vsock->send_pkt_queue, skb)
         -> queue_work(virtio_vsock_workqueue, &vsock->send_pkt_work)
   sock->state = SS_CONNECTING;
```
* Qemu 使用的 `vmw_vsock_virtio_transport.ko` 驱动 probe 时初始化发包的工作队列 `vsock->send_pkt_work`
```c
//net/vmw_vsock/virtio_transport.c
virtio_vsock_probe()
-> INIT_WORK(&vsock->send_pkt_work, virtio_transport_send_pkt_work)
```
* 因此发包时是通过 `virtio_transport_send_pkt_work() -> virtqueue_add_sgs()` 往 virtqueue 写数据

## References
* [Understanding Vsock. General information](https://medium.com/@F.DL/understanding-vsock-684016cf0eb0)
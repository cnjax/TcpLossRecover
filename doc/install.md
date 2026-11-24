# 安装

一台 Linux 服务端，加上一个或多个客户端（Linux、macOS、Windows）。同一个程序两端通用，带 `-s` 就是客户端，不带就是服务端。

```
浏览器 --socks5/http--> tlr 客户端 (127.0.0.1:1081) ==16 条 TCP 连接==> tlr 服务端 (:9000) --> 目标
```

各参数的含义和取值方法见 [README](../README.md#参数)。

## 1. 下载

从 [bin/](../bin) 下载对应的文件：

| 系统 | 文件 |
|---|---|
| Linux x86_64 | `tlr-linux-amd64` |
| Linux aarch64 | `tlr-linux-arm64` |
| Linux 32 位 ARM（armv7，如 OpenWrt 的 ipq40xx 路由器；`uname -m` 显示 `armv7l`） | `tlr-linux-armv7` |
| macOS Apple 芯片 | `tlr-darwin-arm64` |
| macOS Intel | `tlr-darwin-amd64` |
| Windows | `tlr-windows-amd64.exe` |

两端必须用同一版本的文件。

## 2. 生成密钥

```sh
openssl rand -hex 16
```

两端用同一个密钥（`-k`）。隧道的加密和双向认证都靠它，不要用默认值，也不要用好猜的词。

## 3. 服务端（Linux）

以下命令都用 root 执行。

### 3.1 放好程序

把对应架构的文件拷到服务器（`uname -m` 显示 `x86_64` 用 amd64，`aarch64` 用 arm64）：

```sh
chmod +x tlr-linux-amd64
scp tlr-linux-amd64 root@服务器:/usr/local/bin/tlr
```

### 3.2 安装 brutal 模块

下载方向由服务端发送，速度靠这个内核模块。要求内核 5.10 以上、模块 2.0 以上：

```sh
apt install -y git make gcc linux-headers-$(uname -r)
git clone https://github.com/apernet/tcp-brutal && cd tcp-brutal && make
mkdir -p /lib/modules/$(uname -r)/extra
cp brutal.ko /lib/modules/$(uname -r)/extra/ && depmod -a
echo brutal > /etc/modules-load.d/brutal.conf      # 开机自动加载
modprobe brutal && lsmod | grep brutal             # 现在加载并确认
```

- 模块只影响主动选它的连接，也就是 tlr 的隧道连接，系统里其他程序不受影响。
- 内核升级后要重新编译并拷贝一次。
- 装不了模块（例如内核太旧、容器里）时，用 3.3 里"没有 brutal"的那一行，速度会低一些。

### 3.3 注册成系统服务

写入 `/etc/systemd/system/tlr.service`，把 `密钥` 换成第 2 步生成的：

```ini
[Unit]
Description=tlr tunnel server
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/tlr -l :9000 -k 密钥 -rate 800 -life 0 -w 8192 -cc brutal
Restart=always
RestartSec=2
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

没有 brutal 模块时，`ExecStart` 换成：

```
ExecStart=/usr/local/bin/tlr -l :9000 -k 密钥 -rate 800
```

然后启动并设为开机自启：

```sh
chmod 600 /etc/systemd/system/tlr.service
systemctl daemon-reload
systemctl enable --now tlr
```

- Linux 上 `-rate` 是上限：服务端会在它以下自己找链路承受得了的速率，brutal 的发送速率也跟着调。设成不低于链路线速即可，`-rate 800` 是测试链路的值，详见 [怎么定 -rate](../README.md#怎么定--rate)。
- 服务要以 root 运行（上面的写法就是）。不是 root 时，接收缓冲会被 `net.core.rmem_max` 限制住，而且不报错，只是速度上不去。
- 密钥写在命令行里，这台机器上的其他用户能通过进程列表看到它。服务器上有其他人登录时要注意。

### 3.4 放行端口

在云厂商的安全组和本机防火墙里放行 TCP 9000（入方向）。隧道只用 TCP，不需要放行 UDP。

### 3.5 确认

```sh
systemctl is-active tlr          # 应显示 active
journalctl -u tlr -n 20 -o cat   # 看日志
```

日志里出现下面这些说明有问题：

| 日志 | 原因 |
|---|---|
| `warning: default -k` | 没设密钥。 |
| `set congestion control "brutal": ...` | 模块没加载，回到 3.2。 |
| `set brutal params: ...` | 模块版本低于 2.0。 |
| `bad preamble from ...` | 有客户端用了不同的密钥，或者是别的东西在扫这个端口。 |
| `TCP memory at the host's limit (...)` | 服务器的 TCP 内存到了内核的压力线，隧道发不快，见 3.6。 |

### 3.6 内存 1 GB 左右的服务器

内核给全部 TCP 连接的内存设了一条压力线：`net.ipv4.tcp_mem` 的中间那个数，单位是页（一页 4 KB）。默认值按内存大小算，1 GB 内存时约 13700 页，即 54 MB。隧道连接多、往返时间长、丢包重时，等待确认的数据会把它占满，内核就不再让连接排更多数据：速度上不去，丢包却不高。

实测：一台 1 GB 内存的服务端，12 条运下载的连接，丢包约 10%，RTT 190 ms，出口流量在约 500 Mbit/s 处封顶；调高压力线之后，同样的下载从 365–395 Mbit/s 升到 475–510 Mbit/s。

看是不是这种情况：

- 服务端日志里有 `TCP memory at the host's limit (55 of 54 MB): ...`。
- 下载时 `cat /proc/net/sockstat` 里 `TCP:` 那一行的 `mem`（页）接近 `sysctl net.ipv4.tcp_mem` 的中间值。

调高（这组值在 1 GB 内存的机器上用过，单位是页，约合 117、160、200 MB）：

```sh
echo 'net.ipv4.tcp_mem = 30000 41000 51200' > /etc/sysctl.d/90-tlr-tcpmem.conf
sysctl -p /etc/sysctl.d/90-tlr-tcpmem.conf
```

内存更大的机器默认值已经够高，不用改。

## 4. 客户端

客户端监听 `127.0.0.1:1081`，这一个端口同时是 SOCKS5 代理（含 UDP）和 HTTP 代理。

### 4.1 macOS 和 Linux

```sh
chmod +x tlr-darwin-arm64
./tlr-darwin-arm64 -s 服务器地址:9000 -l 127.0.0.1:1081 -k 密钥 \
    -n 16 -rate 800 -life 0 -w 8192 -rcvbuf 16777216
```

- `-n 16` 是起始的连接数，连上 4 条就能用，其余在后台继续拨。服务端发现每条连接本身是瓶颈时，会让客户端把连接加到 `-nmax`（默认 64）为止，没用就退回。
- macOS 第一次运行从别处拷来的文件可能被系统拦截，执行一次 `xattr -d com.apple.quarantine ./tlr-darwin-arm64`。
- Linux 客户端要长期运行，照 3.3 写一个服务，把 `ExecStart` 换成上面这条命令。

### 4.2 Windows

把 `tlr-windows-amd64.exe` 拷到任意目录，在命令提示符里运行：

```
tlr-windows-amd64.exe -s 服务器地址:9000 -l 127.0.0.1:1081 -k 密钥 -n 16 -rate 800 -life 0 -w 8192 -rcvbuf 16777216
```

窗口开着就在运行，关掉窗口即停止。第一次运行时防火墙可能弹出提示，客户端只监听本机地址，不放行也能用。

**Windows 版只验证过能编译，还没有在真机上运行过。**

### 4.3 两端要一致的参数

| 参数 | 要求 |
|---|---|
| `-k` | 两端相同。 |
| `-w` | 两端相同。 |
| 程序版本 | 两端相同。 |
| `-rate`、`-fec`、`-cc`、`-lane` | 各端只管自己发出的方向，不需要相同。 |

macOS 和 Windows 客户端设不了拥塞控制，也测不了丢包：上传方向用系统默认的拥塞控制，`-fec auto` 固定在最强档，`-rate` 是固定限速，不会自动找速率。下载方向由服务端决定，不受影响。

## 5. 让程序走代理

| 用法 | 设置 |
|---|---|
| 浏览器或应用里填代理 | SOCKS5 或 HTTP，地址 `127.0.0.1`，端口 `1081`。 |
| 系统代理 | 系统设置里的 HTTP 和 HTTPS 代理都填 `127.0.0.1:1081`。 |
| 局域网共享 | 客户端的 `-l` 改成 `0.0.0.0:1081`。代理没有认证，只在可信的局域网里这样做。 |
| v2rayN | 添加一个 SOCKS 服务器，地址 `127.0.0.1`，端口 `1081`，tlr 自己另外启动。分流和 TUN 由 v2rayN 负责。这个用法还没有实际验证过。 |

## 6. 验证

在客户端机器上：

```sh
curl -x socks5h://127.0.0.1:1081 https://example.com/ -o /dev/null -w '%{http_code}\n'   # SOCKS5
curl -x http://127.0.0.1:1081   https://example.com/ -o /dev/null -w '%{http_code}\n'   # HTTP 代理
```

都应输出 `200`。客户端日志里每个请求有一行：

```
example.com:443: down 4.8 KB, up 0.6 KB, 631ms
```

| 现象 | 原因 |
|---|---|
| 日志没有 `tunnel up`，反复出现 `tunnel: ...` | 连不上服务端：地址或端口不对、端口没放行、服务端没启动。`tunnel: ... conns in 30s` 是 30 秒内连上的不到 4 条，链路很差。 |
| `tunnel: ... server does not hold our key` | 两端密钥不一致。 |
| 请求打出 `refused ...` | 隧道还没连上。 |
| 请求很快结束且 `down 0.0 KB` | 服务端连不上目标网站。 |
| macOS 客户端日志有 `set receive buffer to ...` | `-rcvbuf` 超过了系统上限 `kern.ipc.maxsockbuf`，调低 `-rcvbuf` 或调高这个上限。 |
| 能用但很慢 | 看服务端日志有没有 3.5 表里的报错；确认两端 `-w` 相同；服务器内存小的见 3.6。 |

## 7. 升级

两端都要换，先换服务端：

```sh
scp tlr-linux-amd64 root@服务器:/usr/local/bin/tlr.new
ssh root@服务器 'mv /usr/local/bin/tlr.new /usr/local/bin/tlr && systemctl restart tlr'
```

重启服务端会断开所有连接，客户端自动重连。然后换客户端的文件并重启客户端。

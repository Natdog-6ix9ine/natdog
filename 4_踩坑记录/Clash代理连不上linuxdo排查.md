# 为什么开了 Clash 系统代理反而连不上 linux.do？

> 一次从 DNS 污染到 TCP 干扰的完整排查过程

---

## 背景与问题描述

[linux.do](https://linux.do/) 是一个部署在 Cloudflare 上的论坛，在国内直连访问速度不稳定。自然的想法是把它加入 Clash 的直连规则，让它走本地网络绕过代理。

但实际操作后发现了一个奇怪的现象：

|情况|结果|
|---|---|
|不开 Clash，直接访问|✅ 稳定能连上|
|开启 Clash，配置直连规则|❌ 经常连不上，ERR_CONNECTION_CLOSED|
|开启 Clash TUN 模式|✅ 能连上|
|开启 Clash，走代理节点|✅ 能连上|

同样是"直连"，为什么经过 Clash 系统代理就不行？下面记录完整的排查过程。

---

## 第一步：怀疑 DNS 污染

Chrome 里配置了 DoH（DNS over HTTPS）之后能访问，这是 DNS 污染的典型特征。用 `nslookup` 对比验证：

```
# 运营商 DNS
$ nslookup linux.do
服务器: zte.home
名称: linux.do
Address: 128.242.245.221   ← 污染的 IP，连不上

# 1.1.1.1
$ nslookup linux.do 1.1.1.1
服务器: one.one.one.one
名称: linux.do
Address: 154.83.15.20      ← 真实的 Cloudflare IP
```

两个 IP 完全不同，确认运营商 DNS 存在污染。

但通过 `chrome://net-internals/#dns` 检查，发现 Chrome 实际上拿到了正确的 IP（`104.20.16.234`）。说明 **Chrome 的 DoH 在系统代理模式下依然生效**，DNS 解析不是连不上的根本原因。

---

## 第二步：Fake-IP 模式的坑

Clash 默认使用 Fake-IP 模式。在这个模式下，所有域名解析都返回一个假 IP（`198.18.x.x` 段），真正的连接由 Clash 内核在匹配规则后发起。

如果配置了 `DIRECT` 规则但没有把域名加入 Fake-IP 过滤列表，Clash 会拿着假 IP 去直连，自然连不上任何东西。解决方法是加入「真实 IP 回应」列表：

```yaml
dns:
  fake-ip-filter:
    - "+.linux.do"
```

或者直接在 ClashParty 的 DNS 设置界面手动添加 `+.linux.do`。

加了之后，问题依然存在——说明还有其他原因。

---

## 第三步：TUN 模式能连，系统代理不行

对比两种模式后发现，TUN 模式下直连完全正常，只有系统代理模式下直连失败。

一个有趣的现象：系统代理直连失败，但**走代理节点反而成功**。

原因在于走代理节点时，本机发出的是加密的代理协议流量，目标是代理服务器，运营商看不出你在访问 linux.do，自然也无法干扰。

---

## 第四步：控制变量实验

为了精确定位，需要排除 DNS 污染的干扰。用 `--resolve` 参数强制指定正确 IP：

```bash
# 实验一：走 Clash 系统代理，指定正确 IP
curl -v --proxy http://127.0.0.1:4356 \
     --resolve linux.do:443:154.83.15.20 \
     https://linux.do
```

结果：

```
HTTP/1.1 200 Connection established   ← TCP 连接成功
schannel: failed to receive handshake ← TLS 握手失败
```

```bash
# 实验二：完全直连，指定正确 IP
curl -v --noproxy "*" \
     --resolve linux.do:443:154.83.15.20 \
     https://linux.do
```

结果：

```
connect failed: Timed out  ← TCP 层就超时了（IP 被运营商丢包）
```

换 Chrome 解析出的另一个 IP `104.20.16.234` 直连测试：

```
Connection aborted, ConnectionResetError(10054)  ← TCP RST，连接被主动重置
```

**两个独立问题同时存在：**

- `154.83.15.20` 在当前网络环境被丢包（IP 不可达）
- `104.20.16.234` 直连会被 TCP RST

---

## 第五步：TLS 握手失败的根因

用 Python 手动模拟 HTTP CONNECT 隧道，然后在隧道上做 TLS 握手：

```python
import socket, ssl

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(('127.0.0.1', 4356))
s.send(b'CONNECT linux.do:443 HTTP/1.1\r\nHost: linux.do:443\r\n\r\n')
resp = s.recv(1024)
# → b'HTTP/1.1 200 Connection established'  ✅ 隧道建立成功

ctx = ssl.create_default_context()
ctx.check_hostname = False
ctx.verify_mode = ssl.CERT_NONE
tls = ctx.wrap_socket(s, server_hostname='linux.do')
# → ssl.SSLEOFError: EOF occurred in violation of protocol  ❌
```

`SSLEOFError: EOF occurred in violation of protocol` —— TLS 握手过程中对方直接发送了 EOF，而不是正常的握手响应。

这是 **SNI 阻断**的典型特征：中间设备检测到 TLS ClientHello 里的 `server_hostname=linux.do`，在握手中途直接切断连接。

---

## 第六步：TLS 指纹是真正原因吗？

既然不开 Clash 时 Chrome 能访问，是否是因为经过 Clash 系统代理转发后 TLS 指纹发生了变化？

使用 `curl_cffi` 库，它底层使用 BoringSSL，能完整模拟 Chrome 的 TLS ClientHello 指纹：

```python
from curl_cffi import Curl, CurlOpt

c = Curl()
c.setopt(CurlOpt.URL, b'https://linux.do/')
c.setopt(CurlOpt.RESOLVE, [b'linux.do:443:104.20.16.234'])  # 正确 IP
c.perform()
# → curl: (35) Recv failure: Connection was reset  ❌
```

**即使使用 Chrome TLS 指纹直连，仍然被 TCP RST。**

对比汇总：

| 测试方式               | TLS 指纹  | 经过 Clash | 结果    |
| ------------------ | ------- | -------- | ----- |
| Python ssl 直连      | OpenSSL | 否        | ❌ RST |
| curl_cffi 直连       | Chrome  | 否        | ❌ RST |
| 不开 Clash，Chrome 直连 | Chrome  | 否        | ✅ 成功  |
| Clash TUN 模式 + 直连  | Chrome  | 是（TUN）   | ✅ 成功  |

最关键的对比是：**同样是 Chrome 指纹 + 正确 IP + 不经过代理，curl_cffi 被 RST，而 Chrome 浏览器成功。**

这说明问题不在 TLS 指纹，而在于更底层的 TCP 特征差异。浏览器和命令行工具在 TCP 窗口大小、timestamp、MSS 等参数上存在差异，运营商的检测是基于这些参数的**概率性模型**，所以有时候能连上，有时候不行。

---

## 结论

这个问题涉及多个叠加因素：

1. **DNS 污染**：运营商 DNS 对 linux.do 返回错误 IP，需要配合 DoH 或 Clash 的可靠 DNS 才能正确解析
2. **Fake-IP 干扰**：Clash Fake-IP 模式下，直连规则需要配合 `fake-ip-filter` 才能正常工作
3. **TCP 层概率性干扰**：经过 Clash 系统代理转发的直连流量，底层 TCP 特征与原生浏览器不同，触发了运营商的干扰，但不是100%阻断，所以有时候能连上
4. **TUN 模式**绕过了所有问题：在网络层接管流量，行为与原生系统栈更接近，同时自带可靠的 DNS 解析

**实际解决方案：使用 TUN 模式，或将 linux.do 配置为走代理节点而非直连。**

---

_实验环境：Windows 11 / ClashParty v1.19.20 / Mihomo 内核 / 中国电信宽带_
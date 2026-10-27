# 自建 DERP 中继部署清单（国内加速 Tailscale）

> 场景：Tailscale 打洞失败时走海外官方中继（国内 300ms+）。
> 自建一台国内中继，兜底延迟从 300ms 降到 20~40ms。
> 前提：一台国内云服务器（**购买前确认 UDP 入站未被封**——部分机房整体拦 UDP，安全组开了也没用，用 `tcpdump` 实测）。

## 前置检查

```bash
# 1. TCP 入站连通性（部署后从外部测）
openssl s_client -connect 服务器IP:8443 -servername derp.example.com

# 2. UDP 入站实测（服务器上抓包 + 外部发包，0 包 = 机房拦 UDP，换机器）
tcpdump -i any udp port 3478 -nn          # 服务器：抓包
# 外部：任意 UDP 客户端发 3478 端口，看抓包结果

# 3. hairpin 检查：服务器访问自己的公网 IP 是否可达（多数机房不通）
#    不通就用 /etc/hosts 把域名指向 127.0.0.1 绕行
```

## 自签证书（配合 Tailscale ACL 的 InsecureForTests）

```bash
mkdir -p /opt/derper/certs
openssl req -x509 -newkey rsa:2048 -nodes -days 3650 \
  -keyout /opt/derper/certs/derp.example.com.key \
  -out /opt/derper/certs/derp.example.com.crt \
  -subj "/CN=derp.example.com"
```

## derper 二进制

```bash
# 方式一：有 Go 环境
go install tailscale.com/cmd/derper@latest
# 方式二：从已有部署直接拷贝二进制（同架构即可，实测可行）
```

安装 [derper.service](derper.service)（改域名与路径）后：

```bash
systemctl daemon-reload && systemctl enable --now derper
# 验证：必须带正确 SNI 测试（无 SNI 会被 derper 拒绝）
curl -sk --resolve derp.example.com:8443:127.0.0.1 https://derp.example.com:8443/generate_204
# 期望: [204]
```

## 注册到你的 tailnet

login.tailscale.com → **Access Controls** → 加入 derpMap（HostIp 换成你的服务器 IP）：

```jsonc
"derpMap": {
  "Regions": {
    "900": {
      "RegionID":   900,
      "RegionCode": "myderp",
      "RegionName": "My DERP",
      "Nodes": [{
        "Name":     "1",
        "RegionID": 900,
        "HostName": "derp.example.com",
        "DERPPort": 8443,
        "IPv4":     "服务器IP",
        "InsecureForTests": true,   // 自签证书必须
      }],
    },
  },
},
```

## 验证

```bash
# 任一 tailnet 设备上
tailscale netcheck        # 期望: Nearest DERP 是你的 RegionCode，且有延迟数据
tailscale ping <对端IP>   # 打洞失败时应 via DERP(myderp) 而非海外节点
```

## 安全注意

- `InsecureForTests` 跳过的只是 **客户端对 derper 的 TLS 校验**；设备间数据仍是 WireGuard
  端到端加密——derper 只能搬运密文，无法解密内容。
- 想更严可给 derper 加 `-verify-clients`（要求 derper 所在机器已加入同一 tailnet）。

# nas-from-zero 🍮

> 从零到家庭数据中心：绿联 DH4300 Plus + UGOS 48 小时实录，含全部踩坑与可复用脚本。

**最终成果**：相册自动备份 · Jellyfin 4K 直连影音库 · 阿里云盘挂载 · Tailscale 远程访问（自建国内 DERP 中继，延迟 300ms → 24ms）· RAID1 + 快照 + PC 自动备份。

## 硬件与架构

```
[绿联 DH4300 Plus]           [云服务器 x2]
  RAID1 (Btrfs)                ├── derper 自建中继 (TCP 8443)
  UGOS Pro                     └── Tailscale 节点 (UDP 41641)
  Jellyfin (host 网络)
  Alist / 迅雷 / 快照           [客户端]
  Tailscale (host 网络)  ←────  iPhone (Vidhub/Tailscale)
  SMB → Windows Z 盘            Windows PC (SMB Z 盘 + 计划任务备份)
```

## 目录导航

| 路径 | 内容 |
|---|---|
| [docs/lessons.md](docs/lessons.md) | **10 大踩坑实录**（精华，先看这个） |
| [jellyfin/](jellyfin/) | Jellyfin Docker Compose + API 初始化流程 |
| [tailscale/](tailscale/) | host 模式 Compose（含 bridge 反面教材对比） |
| [derper/](derper/) | 自建 DERP 中继：systemd 服务 + 部署清单 |
| [backup/](backup/) | Windows → NAS 每日增量备份脚本（robocopy） |

## 核心原则（血泪换来的）

1. **NAS 上宁缺毋滥**：一个应用一个端口，同类应用绝不二装
2. **远程访问只走加密隧道**：Tailscale/WireGuard，永不裸奔端口映射
3. **备份必须演练**：没恢复过的备份 = 没备份
4. **快照 ≠ 备份**：快照防手滑，备份防硬盘坏，两件事
5. **确定性 > 概率**：自建 DERP 把「打洞大概率高」变成「100% 可用」

## 安全模型

- 数据面：WireGuard 端到端加密（现代 VPN 密码学标准）
- 准入面：仅账号下已授权设备，无万能密钥遗留
- 暴露面：公网 0 开放端口（Tailscale UDP 打洞端口除外，无密钥握手必失败）
- 中继面：自建 DERP 只搬运密文，无法解密内容

## 环境说明

- 适用：UGOS Pro（绿联/其他 Btrfs NAS）+ 任意 x86 云服务器
- 不适用：软路由/虚拟机玩法（ARM NAS 带不动）
- 所有脚本不包含任何凭据，凭据一律运行时输入或自行填写

## License

MIT

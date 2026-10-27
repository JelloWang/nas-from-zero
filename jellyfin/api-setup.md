# Jellyfin API 初始化流程（10.10.x）

> 全新配置目录启动后，向导未完成状态可全 API 走完，无需浏览器。
> 注意：**Jellyfin 12.x 改了 /Startup/User 端点**，本流程仅适用 10.x。

```bash
JF=http://NAS内网IP:8096

# 1. 语言与元数据区域
curl -s -X POST "$JF/Startup/Configuration" \
  -H "Content-Type: application/json" \
  -d '{"UICulture":"zh-CN","MetadataCountryCode":"CN","MetadataLanguage":"zh-CN"}'

# 2. 创建管理员（字段是 Password 不是 Pw！）
curl -s -X POST "$JF/Startup/User" \
  -H "Content-Type: application/json" \
  -d '{"Name":"你的用户名","Password":"你的密码"}'

# 3. 确认第 2 步成功（200/204）后再完成向导 —— 跳步会锁死
curl -s -X POST "$JF/Startup/Complete"

# 4. 登录换 token
TOKEN=$(curl -s -X POST "$JF/Users/AuthenticateByName" \
  -H "Content-Type: application/json" \
  -H 'X-Emby-Authorization: MediaBrowser Client="setup", Device="setup", DeviceId="setup1", Version="1.0"' \
  -d '{"Username":"你的用户名","Pw":"你的密码"}' | jq -r .AccessToken)

# 5. 建媒体库（电影）
curl -s -X POST "$JF/Library/VirtualFolders?name=电影&collectionType=movies&paths=/media/电影&refreshLibrary=false" \
  -H "X-Emby-Token: $TOKEN"

# 6. 建媒体库（剧集）
curl -s -X POST "$JF/Library/VirtualFolders?name=剧集&collectionType=tvshows&paths=/media/剧集&refreshLibrary=false" \
  -H "X-Emby-Token: $TOKEN"

# 7. 触发扫描
curl -s -X POST "$JF/Library/Refresh" -H "X-Emby-Token: $TOKEN"

# 8. 验证
curl -s "$JF/Library/VirtualFolders" -H "X-Emby-Token: $TOKEN"
```

## 插件安装（OpenSubtitles 为例）

```bash
# 从服务端自带目录查包（含版本与兼容 ABI）
curl -s "$JF/Packages?targetSystem=System" -H "X-Emby-Token: $TOKEN" | jq '.[] | select(.name | contains("Subtitles"))'

# 安装（服务端自己去官方源下载）
curl -s -X POST "$JF/Packages/Installed/Open%20Subtitles?assemblyGuid=插件GUID&version=20.0.0.0" \
  -H "X-Emby-Token: $TOKEN"

# 重启生效
curl -s -X POST "$JF/System/Restart" -H "X-Emby-Token: $TOKEN"
```

## 锁死自救（向导完成但无账号）

```bash
# 停容器 → 把 config/data 下的 jellyfin.db / library.db 改名（不删除，留后路）→ 起容器 → 重跑上面流程
mv /path/to/config/data/jellyfin.db /path/to/config/data/jellyfin.db.bak
```

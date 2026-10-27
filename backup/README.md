# Windows → NAS 每日增量备份（robocopy）

> 零软件方案：Windows 计划任务 + robocopy，增量复制、**只增不删**（本地误删 NAS 还有底）。

## backup 脚本（pc-backup.ps1）

```powershell
$ErrorActionPreference = 'Continue'
# 按需增删源目录；目标为 NAS 的 SMB 共享
robocopy 'C:\Users\你的用户名\Desktop\知识库' '\\NAS内网IP\PC备份\知识库' /E /XO /R:1 /W:1 /NFL /NDL /NP /LOG+:'D:\NASBackup\backup.log'
robocopy 'D:\工作区' '\\NAS内网IP\PC备份\workbuddy' /E /XO /R:1 /W:1 /NFL /NDL /NP /LOG+:'D:\NASBackup\backup.log'
```

参数说明：`/E` 含子目录 · `/XO` 只拷新的（增量）· `/R:1 /W:1` 失败不重试 · `/LOG+` 追加日志。
**注意没有 `/MIR`**——镜像模式会同步删除，备份场景慎用。

## 注册计划任务（每日 23:00）

脚本存 `D:\NASBackup\pc-backup.ps1`，然后：

```powershell
$action = New-ScheduledTaskAction -Execute 'powershell.exe' -Argument '-NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -File D:\NASBackup\pc-backup.ps1'
$trigger = New-ScheduledTaskTrigger -Daily -At 23:00
$settings = New-ScheduledTaskSettingsSet -StartWhenAvailable -DontStopOnIdleEnd
Register-ScheduledTask -TaskName 'NAS_PCBackup' -Action $action -Trigger $trigger -Settings $settings
```

## 验证（备份的最后一课：没恢复过的备份 = 没备份）

1. 首次全量后，到 NAS 共享文件夹确认文件齐全
2. **恢复演练**：从 NAS 随便挑一个文件拷回本地临时目录，能打开 = 备份体系可信
3. 每月看一眼 `D:\NASBackup\backup.log` 有无异常

## 安全边界

- NAS 侧共享文件夹默认带回收站（误覆盖可回滚）
- 敏感数据（证件扫描等）建议单独共享文件夹并关闭外部访问权限
- 备份 ≠ 冗余：RAID/快照防手滑和单盘故障，异地/多介质才是完整 3-2-1

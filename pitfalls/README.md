# 踩坑记录

格式：日期 / 现象 / 原因 / 解法。每坑一个文件，文件名 `YYYY-MM-DD-关键词.md`。

## 2026-09-27 aka.ms 不可达导致微软安装器全部失败

- **现象**：VS Build Tools 安装器报 `Error 0x80131500 无法下载通道清单`，退出码 1。
- **原因**：`aka.ms` 托管在 Akamai CDN（2.23.246.9 等），国内无节点，TLS 握手被掐断；而 `download.visualstudio.microsoft.com` 有移动/电信国内节点，直连正常。
- **解法**：用 DoH 拿到能握手的 Akamai 节点，顺着 `aka.ms/vs/17/release/channel` 的 301 拿到直连清单 URL，喂给安装器：
  `vs_BuildTools.exe --channelUri "<直连 chman URL>" --add Microsoft.VisualStudio.Workload.VCTools --includeRecommended`
  全程不碰 aka.ms，8 分钟装完。清单副本备份在 `D:\download\VSBuildTools\`。

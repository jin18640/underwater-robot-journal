# 踩坑记录

格式：日期 / 现象 / 原因 / 解法。每坑一个文件，文件名 `YYYY-MM-DD-关键词.md`。

## 2026-09-27 aka.ms 不可达导致微软安装器全部失败

- **现象**：VS Build Tools 安装器报 `Error 0x80131500 无法下载通道清单`，退出码 1。
- **原因**：`aka.ms` 托管在 Akamai CDN（2.23.246.9 等），国内无节点，TLS 握手被掐断；而 `download.visualstudio.microsoft.com` 有移动/电信国内节点，直连正常。
- **解法**：用 DoH 拿到能握手的 Akamai 节点，顺着 `aka.ms/vs/17/release/channel` 的 301 拿到直连清单 URL，喂给安装器：
  `vs_BuildTools.exe --channelUri "<直连 chman URL>" --add Microsoft.VisualStudio.Workload.VCTools --includeRecommended`
  全程不碰 aka.ms，8 分钟装完。清单副本备份在 `D:\download\VSBuildTools\`。

## 2026-09-27 WSL 打不开：LxssManager 服务被禁用

- **现象**：打开 Ubuntu，窗口一闪就关。
- **原因**：WSL 引擎服务 `LxssManager` 的启动类型被设成 Disabled。DISM 开功能救不了（功能本来就开着）。
- **解法**：管理员终端改注册表 `Start` 为 `3` 后重启；或 `Set-Service -Name LxssManager -StartupType Manual`。
  详情见 [2026-09-27-LxssManager被禁用.md](2026-09-27-LxssManager被禁用.md)。

## 2026-09-27 OpenFOAM 污染 LD_LIBRARY_PATH，VS Code 远程连不上

- **现象**：`code .` 时 node 报 `CXXABI_1.3.11 not found`。
- **原因**：`.bashrc` 里全局 source 了 OpenFOAM，把它自带的 2019 年 gcc 6.3.0 库塞进 `LD_LIBRARY_PATH`。
- **解法**：临时 `env -u LD_LIBRARY_PATH code .`；治本是把 source 那行改成 `of()` 函数按需加载。
  详情见 [2026-09-27-OpenFOAM污染LD_LIBRARY_PATH.md](2026-09-27-OpenFOAM污染LD_LIBRARY_PATH.md)。

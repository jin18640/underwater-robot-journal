# 2026-09-27 WSL 打不开：LxssManager 服务被禁用

- **现象**：开始菜单打开 Ubuntu，窗口一闪就自动关闭，没有任何报错。
- **原因**：WSL 的引擎服务 `LxssManager` 被设成了 **Disabled / Stopped**（注册表
  `HKLM\SYSTEM\CurrentControlSet\Services\LxssManager` 的 `Start=4`）。
  引擎没起来，Ubuntu 自然没人接管。Ubuntu 应用包本身是好的（磁盘 5.1 GB 说明以前跑起来过），
  多半是某次"系统优化"或装完大软件后为了省内存手动关的。
- **解法**（需管理员）：
  1. 管理员 PowerShell：`Set-Service -Name LxssManager -StartupType Manual; Start-Service -Name LxssManager`
  2. 若报"拒绝访问"：说明窗口没真正提权。用 `Win + R` 输入 `powershell` 后按 `Ctrl + Shift + Enter`，
     必须看到蓝色的 UAC 确认框并点"是"。
  3. 即便提权成功仍被拒（服务安全描述符被改过）：绕过服务管理器直接改注册表 ——
     `reg add "HKLM\SYSTEM\CurrentControlSet\Services\LxssManager" /v Start /t REG_DWORD /d 3 /f`
     （`3` = 手动启动），**重启**后生效。我就是靠这条修好的。
- **误区**：`dism /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux` 没用 ——
  WSL 功能本来就开着，DISM 只回一句"功能已启用"，不会去改一个被手工禁用的服务启动类型。

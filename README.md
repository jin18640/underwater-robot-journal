# 水下机器人工程日志

从零制作一台水下机器人（ROV）的完整记录。技术主线：[FinsROV](https://github.com/FPJ-GAOGE/FinsROV-An-Underwater-Camera-Based-Multi-Robot-Platform)（上海交大 FINS 实验室方案，MIT 协议）。

## 目录结构

- `notes/` —— 选型分析与项目解读、本机环境清单
- `progress/` —— 按日期记录的进度
- `pitfalls/` —— 踩坑记录（现象 / 原因 / 解法）

## 环境

- OS: Windows 11 + WSL2 (Ubuntu 24.04 LTS)
- 编辑器: VS Code 1.139.1（C/C++、Python、CMake、Remote-WSL；Windows 与 WSL 两侧分别装）
- 工具链: Windows 侧 MSVC 14.44 与 MinGW-w64 GCC 16.1 双轨；WSL 侧 gcc 13.3.0 + gdb + cmake + ninja
- 仿真: OpenFOAM v2012（装在 WSL，敲 `of` 进入环境）
- 版本管理: Git 2.55 + GitHub CLI 2.101
- 待装: STM32CubeCLT（下位机开工前）、ROS 2 Jazzy（上位机开工前）

完整清单见 [notes/本机开发环境清单.md](notes/本机开发环境清单.md)。

## 进度速览

| 日期 | 事项 |
|---|---|
| 2026-09-27 | 开坑。Windows 与 WSL 双侧开发环境搭建完成；修复打不开的 WSL；fork FinsROV；本仓库建立 |

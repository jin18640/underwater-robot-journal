# 水下机器人工程日志

从零制作一台水下机器人（ROV）的完整记录。技术主线：[FinsROV](https://github.com/FPJ-GAOGE/FinsROV-An-Underwater-Camera-Based-Multi-Robot-Platform)（上海交大 FINS 实验室方案，MIT 协议）。

## 目录结构

- `notes/` —— 选型分析与项目解读
- `progress/` —— 按日期记录的进度
- `pitfalls/` —— 踩坑记录（现象 / 原因 / 解法）

## 环境

- OS: Windows 11 + WSL2 (Ubuntu)
- 编辑器: VS Code（C/C++、Python、CMake、Remote-WSL、STM32 按需 Profile）
- 工具链: MSVC 14.44（Build Tools 2022）与 MinGW-w64 GCC 16.1 双轨；STM32CubeCLT 待装
- 版本管理: Git 2.55 + GitHub CLI 2.101

## 进度速览

| 日期 | 事项 |
|---|---|
| 2026-09-27 | 开坑。开发环境搭建完成；fork FinsROV；本仓库建立 |

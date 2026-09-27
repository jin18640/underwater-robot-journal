# 2026-09-27 OpenFOAM 污染环境变量，导致 VS Code 远程连不上

- **现象**：在 Ubuntu 里敲 `code .`，VS Code Server 下载解压一切正常，最后启动 node 时炸：
  `node: ...gcc-6.3.0/lib64/libstdc++.so.6: version CXXABI_1.3.11 not found`
- **原因**：`~/.bashrc` 里有一行 `source ~/OpenFOAM/OpenFOAM-v2012/etc/bashrc`，
  OpenFOAM 把它自带的 gcc 6.3.0（2019 年）的库路径塞进了 `LD_LIBRARY_PATH`。
  每个终端——包括 VS Code 远程服务端——都继承了这份污染，
  于是 node 去加载老 libstdc++ 而不是系统的，缺符号直接挂掉。
  同一个原因还会让 `gcc` 显示成 6.3.0、遮住系统编译器。
- **解法**：
  - 临时：`env -u LD_LIBRARY_PATH code .`
  - 治本：把那行 source 改成按需加载的函数，让平时终端保持干净：

    ```bash
    # 原来的：source /home/jinyuxin/OpenFOAM/OpenFOAM-v2012/etc/bashrc
    of() { source /home/jinyuxin/OpenFOAM/OpenFOAM-v2012/etc/bashrc; }
    ```

    改完重开终端：平时是干净环境，敲 `of` 才进入 OpenFOAM。
- **通用教训**：任何"往 .bashrc 里 source 一整套环境"的软件（OpenFOAM、ROS、conda、CUDA）都会干这种事。
  长期方案是**用函数或 alias 包起来按需加载**，跟 VS Code 用 Profile 隔离 STM32 扩展是同一个思路：
  工具之间别互相污染。
- **补救发现**：Ubuntu 的 Linux 文件系统可以从 Windows 侧直接读写 —— UNC 路径
  `\\wsl.localhost\Ubuntu\home\jinyuxin\`，即使 `wsl.exe` 被安全策略拦截也不影响。
  改 Linux 里的配置文件不必手动 nano。

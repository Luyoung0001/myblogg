---
title: "毕业设计记录（8）：minicom 串口调试"
date: 2026-01-20 10:03:13
tags:
  - "graduation thesis"
  - "minicom"
  - "UART"
categories:
  - "Thesis Project"
---

## 遇到的问题

由于 UART 串口屏幕不工作（可能是线路没接好），只能在仿真环境中运行正常。因此我打算把 UART_TX、UART_RX 的约束引脚绑定从原来的 GPIO 引脚到 FPGA 开发板自带的 CH340E 芯片的引脚上。

CH340E，一个 USB 总线的转接芯片，实现将串口转到 USB接口。这样我就能在下板子之后，将其和电脑 USB 接口连接，电脑上运行的 串口通信软件比如 minicom 就可以接收和解析 FPGA 发来的 UART_TX 数据。

## minicom

minicom 是一个在 Linux/Unix 系统中使用的串口通信程序，类似于 Windows 下的超级终端(HyperTerminal)。

主要用途：
- 通过串口与设备通信（如路由器、交换机、嵌入式设备等）
- 配置网络设备的控制台接口
- 调试串口通信
- 连接到调制解调器

常用功能：
- 支持多种波特率设置
- 提供文件传输协议（如 Xmodem、Ymodem、Zmodem）
- 可以记录会话日志
- 支持自动拨号和脚本功能

基本使用：
### 启动 minicom
minicom

### 指定串口设备启动
minicom -D /dev/ttyUSB0

### 配置 minicom
minicom -s

它是嵌入式开发、网络设备管理等场景中的常用工具。

### 安装 minicom（如果没有）
```bash
sudo apt-get install minicom
```
### 配置串口权限（只需执行一次）
```bash
  sudo usermod -a -G dialout $USER
```
### 打开串口监听（9600波特率）
```bash
  minicom -D /dev/ttyUSB0 -b 9600
```
minicom 快捷键：
- Ctrl+A Z - 帮助菜单
- Ctrl+A X - 退出

## 之后的解决方案

下板子之后，利用 monicom 进行进行显示：

```txt
========================================
  xOS - Simple Operating System
  for LoongArch32R SoC
========================================

Type 'help' for available commands.

xos>
```
由于目前键盘输入还不支持，因此xOS 还不能执行任何命令。
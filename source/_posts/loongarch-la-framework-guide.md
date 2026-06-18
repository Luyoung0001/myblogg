---
title: "LA 框架使用指南：远程环境、工程结构与调试入口"
date: 2026-04-12 10:03:13
tags:
  - "LoongArch"
  - "LA framework"
  - "development environment"
categories:
  - "Computer Architecture"
mermaid: true
---

## Windows 安装 VSCode 并通过 Remote SSH 登录服务器

### 1. 安装 VSCode

1. 打开浏览器，访问 [https://code.visualstudio.com](https://code.visualstudio.com)
2. 点击页面上的 **Download for Windows** 按钮，下载安装包（`.exe` 文件）
3. 双击下载好的安装包，按照安装向导操作：
   - 同意许可协议
   - 选择安装路径（默认即可）
   - **勾选以下选项**（推荐全选）：
     - ✅ 将"通过 Code 打开"操作添加到 Windows 资源管理器文件的上下文菜单
     - ✅ 将"通过 Code 打开"操作添加到 Windows 资源管理器目录的上下文菜单
     - ✅ 将 Code 注册为受支持的文件类型的编辑器
     - ✅ 添加到 PATH（重要，方便命令行使用）
4. 点击 **安装**，完成后启动 VSCode

---

### 2. 安装 Remote SSH 插件

1. 打开 VSCode，点击左侧活动栏中的 **扩展图标**（或按 `Ctrl+Shift+X`）
2. 在搜索框中输入 `Remote - SSH`
3. 找到由 **Microsoft** 发布的 **Remote - SSH** 插件，点击 **Install（安装）**
4. 安装完成后，左侧活动栏底部会出现一个 **远程资源管理器** 图标（显示为一个小电脑屏幕）

---

### 3. 配置 SSH 密钥（推荐，免密登录）

为了避免每次连接都输入密码，建议配置 SSH 密钥对。

**在 Windows 本地生成密钥对：**

1. 按 `Win + R`，输入 `cmd`，打开命令提示符（或使用 PowerShell）
2. 输入以下命令生成密钥：
   ```cmd
   ssh-keygen -t rsa -b 4096
   ```
3. 一路回车（默认保存到 `C:\Users\你的用户名\.ssh\id_rsa`，不设置密码短语）
4. 生成完成后，查看公钥内容：
   ```cmd
   type C:\Users\你的用户名\.ssh\id_rsa.pub
   ```
5. 复制输出的公钥内容（一长串以 `ssh-rsa` 开头的字符串）

**将公钥上传到服务器：**

1. 先用密码方式 SSH 登录服务器一次（见第 4 步）
2. 在服务器上执行：
   ```zsh
   mkdir -p ~/.ssh
   echo "粘贴你的公钥内容" >> ~/.ssh/authorized_keys
   chmod 700 ~/.ssh
   chmod 600 ~/.ssh/authorized_keys
   ```

---

### 4. 在 VSCode 中添加 SSH 主机

1. 在 VSCode 中按 `Ctrl+Shift+P` 打开命令面板
2. 输入并选择 `Remote-SSH: Add New SSH Host...`
3. 在弹出的输入框中，输入 SSH 连接命令，格式如下：
   ```
   ssh 用户名@服务器IP地址
   ```
   例如：
   ```
   ssh student@192.168.1.100
   ```
4. 选择将配置保存到 `C:\Users\你的用户名\.ssh\config`（选第一个即可）
5. 右下角会弹出提示，点击 **Open Config** 可以查看或编辑配置文件

SSH 配置文件示例（`~/.ssh/config`）：
```
Host my-server
    HostName 192.168.1.100
    User student
    Port 22
    IdentityFile C:\Users\你的用户名\.ssh\id_rsa
```

> 其中 `Host my-server` 是你给这台服务器起的别名，之后连接时可以直接用这个名字。

---

### 5. 连接到服务器

1. 点击 VSCode 左侧的 **远程资源管理器** 图标
2. 在 **SSH TARGETS** 列表中，找到刚刚添加的服务器（如 `my-server`）
3. 点击服务器名称右侧的 **箭头图标**（Connect to Host in New Window）
4. VSCode 会打开一个新窗口并开始连接：
   - 如果是首次连接，会提示 **确认服务器指纹**，选择 **Continue**
   - 如果没有配置密钥，会提示输入密码，输入后回车
5. 连接成功后，VSCode 窗口左下角会显示绿色的 `SSH: my-server` 标识

---

### 6. 在远程服务器上打开文件夹

连接成功后，你就可以像操作本地文件一样操作服务器上的文件：

1. 点击左侧 **资源管理器图标**（或按 `Ctrl+Shift+E`）
2. 点击 **Open Folder（打开文件夹）**
3. 在弹出的路径框中输入服务器上的目录路径，例如：
   ```
   /home/student/
   ```
4. 点击 **OK**，即可在 VSCode 中浏览和编辑服务器上的所有文件

---

### 7. 使用集成终端

连接远程服务器后，可以直接在 VSCode 中打开服务器的终端：

1. 按 `` Ctrl+` ``（反引号）或点击菜单 **Terminal → New Terminal**
2. 终端会自动连接到服务器，就像直接 SSH 登录一样，可以运行任何命令

---

### 常见问题

| 问题 | 解决方法 |
|------|----------|
| 连接超时 | 检查服务器 IP 是否正确，确认网络能 ping 通服务器 |
| 权限被拒绝（publickey）| 检查公钥是否正确上传，`authorized_keys` 权限是否为 600 |
| 每次连接都要输入密码 | 检查 `~/.ssh/config` 中 `IdentityFile` 路径是否正确 |
| 插件安装失败 | 检查网络，或在 VSCode 中配置代理 |
| 远程资源管理器看不到服务器 | 重新执行 `Remote-SSH: Add New SSH Host` |

---

如果还有其他问题，可以使用搜索引擎。

## 准备
登录到服务器后，大概率是这样的一个界面：
```zsh
You can:

(q)  Quit and do nothing.  The function will be run again next time.

(0)  Exit, creating the file ~/.zshrc containing just a comment.
     That will prevent this function being run again.

(1)  Continue to the main menu.

(2)  Populate your ~/.zshrc with the configuration recommended
     by the system administrator and exit (you will need to edit
     the file by hand, if so desired).

--- Type one of the keys in parentheses ---
```

此时输入 `0` ，会创建一个默认的 zsh 用户配置文件 .zshrc。

## 克隆 LA 并运行 demo

在用户目录下输入：
```zsh
git clone https://gitee.com/luyoung0010/LA.git
```

然后回车，就会克隆 LA 框架。然后 cd 到 LA 目录，直接输入并回车：

```zsh
make all EXP=6
```
就进入了仿真，loongarch 工具链，包括 verilator 都已经安装好了。在使用之前，务必详细阅读整个项目，了解 LA 是在做什么，是怎么工作的，不懂的话可以问 AI。






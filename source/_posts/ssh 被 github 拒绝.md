---
title: ssh 连不上 github：身份认证失败
date: 2025-12-19 15:03:13
tags: github
categories: github
mermaid: true
---

## 问题

我今天从 github 下拉一个仓库的时候，突然报错：

```bash
Connection closed by 20.205.243.166 port 22
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```

这怎么可能？我已经配置了 ssh，所有数据走的都是 443 端口：

```bash
Host github.com
    HostName ssh.github.com
    Port 443
    User git
```
## 解决方法

直接输入命令就好了：

```bash
ssh-keyscan -p 443 ssh.github.com >> ~/.ssh/known_hosts
# ssh.github.com:443 SSH-2.0-aae3c6b
# ssh.github.com:443 SSH-2.0-aae3c6b
# ssh.github.com:443 SSH-2.0-aae3c6b
# ssh.github.com:443 SSH-2.0-aae3c6b
# ssh.github.com:443 SSH-2.0-aae3c6b
```

### 原理
SSH 首次连接“新“服务器时，会验证服务器的"指纹"（host key），防止中间人攻击。
```txt
  你的电脑 ──SSH连接──> ssh.github.com
                        │
                        ↓
                "我是 GitHub，这是我的指纹: xxx"
                        │
                        ↓
          SSH 检查 ~/.ssh/known_hosts 里有没有这个指纹
                        │
              ┌─────────┴─────────┐
              ↓                   ↓
           没有记录            有记录且匹配
              ↓                   ↓
      "Host key verification   连接成功
           failed"
```
### 命令解释

```bash
ssh-keyscan -p 443 ssh.github.com >> ~/.ssh/known_hosts
```

  | 部分                  | 作用                                          |
  |-----------------------|-----------------------------------------------|
  | ssh-keyscan           | 获取远程服务器的公钥指纹                      |
  | -p 443                | 使用 443 端口（因为我们配置了 GitHub 走 443） |
  | ssh.github.com        | GitHub 的 SSH 服务器地址                      |
  | >> ~/.ssh/known_hosts | 追加到已知主机文件                            |

执行后，`~/.ssh/known_hosts` 里就有了 GitHub 的指纹记录，下次连接就能通过验证。





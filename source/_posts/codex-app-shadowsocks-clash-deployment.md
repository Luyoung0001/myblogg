---
title: "Codex App 执行协议：部署 Shadowsocks 与独立 Clash/Mihomo 订阅"
date: 1970-01-01 00:00:00
tags:
  - "Codex"
  - "Shadowsocks"
  - "Clash"
  - "Mihomo"
  - "Linux"
categories:
  - "Networking"
---

<!--more-->

以下内容直接交给 Codex App 执行。Codex 应自行选择合适的命令、脚本和文件编辑方式；本文只规定输入、目标状态、安全边界和验收标准。

## 1. 先向用户索取原始登录信息

开始任何操作前，必须询问：

```text
VPS IPv4：<必填>
SSH 端口：<默认 22>
初始 SSH 用户名：<必填>
初始 SSH 密码：<必填，通过 Codex 安全输入提供>
目标 sudo 用户名和密码：<仅初始用户为 root 时询问>
本地 Clash 配置路径：<默认 profiles/MyServers.yaml>
第三方规则文件或订阅：<可选，必须是用户有权使用的内容>
```

用户不需要预先生成密钥。Codex 必须自己完成密钥引导，并使后续任务可以通过 SSH 别名直接连接。

密码只能暂存在当前 Codex 任务的安全凭据上下文中，用于首次 SSH 登录、安装公钥、sudo 初始化和本次部署。禁止写入仓库、环境文件、命令参数、shell 历史、日志或最终回答；部署结束后清除。

## 2. 完成 SSH 和 sudo 初始化

使用用户提供的初始用户名和密码建立第一次 SSH 连接，然后自行完成以下目标：

- 在 Codex 执行环境生成 ed25519 密钥，不要求用户提供私钥。
- 将公钥安装到初始登录用户的 `authorized_keys`，私钥只保留在 Codex 执行环境。
- 在 SSH 配置中建立稳定别名，例如 `my-vps`，后续全部通过密钥连接。
- 如果初始用户不是 root，将该用户加入 `sudo` 组并继续使用该账号。
- 如果初始用户是 root，询问并创建一个普通管理用户，将其加入 `sudo` 组，后续改用该账号；root 仅用于引导。
- 验证密钥登录、当前用户、系统版本、CPU 架构和 sudo 权限。权限不足时停止并向用户报告。

完成密钥登录后，继续保留初始密码到所有需要 sudo 的部署步骤结束，再清除密码。不得使用 `sshpass -p` 或任何会把密码暴露在进程列表中的方式。

## 3. 配置 VPS 上的 Shadowsocks 服务

在已建立的密钥 SSH 会话中部署，目标状态如下：

- 系统为 Debian/Ubuntu LTS；按实际 CPU 架构安装 sing-box core。示例固定版本为 `1.13.19`，不得运行第三方仓库的默认 `install.sh`。
- 二进制位于 `~/.local/bin/sing-box`，运行目录为 `~/sing-box-runtime`。
- 服务端配置为 Shadowsocks 入站：监听 `0.0.0.0:40088`，加密方式 `chacha20-ietf-poly1305`，同时支持 TCP 和 UDP，出站为 direct。
- Shadowsocks 密码由 Codex 在 VPS 本地高熵生成，写入权限为 `600` 的文件和配置；不能出现在输出、日志或回答。
- 创建用户级 `sing-box-proxy.service`，启用 linger、开机启动和异常自动重启。
- 实际运行 `sing-box check`，确认服务 active，并确认 `40088/tcp`、`40088/udp` 正在监听。
- 配置 VPS 防火墙放行 `40088/tcp`、`40088/udp` 和订阅服务的 `18080/tcp`。云安全组若无法由 Codex 操作，则提示用户手动放行。

## 4. 修改本地 Clash/Mihomo 配置

以本地工作区的 `profiles/MyServers.yaml` 为独立、可长期保存的 Local profile。若工作区在 VPS，则同时维护远端副本 `~/clash-subscription/profiles/MyServers.yaml`；否则由 Codex 通过已建立的 SSH 连接同步。

### 4.1 节点输入和校验

节点来源可以是结构化字段，也可以是 `ss://` URI。Codex 使用标准 URI/Base64 解析器提取并交叉校验，不要猜测冲突字段，也不要输出解码后的密码：

- `server` 为 VPS IPv4，`port` 为 `40088`；
- 类型为 Shadowsocks，cipher 为 `chacha20-ietf-poly1305`；
- 密码非空，Clash/Mihomo 节点设置 `udp: true`；
- 每个节点名称稳定且唯一，例如 `我的节点-01`、`我的节点-02`。

### 4.2 配置内容

Local profile 必须是完整 Clash/Mihomo YAML，至少包含：

- `mode: rule`；
- 所有个人节点，且只包含个人节点；
- 名为 `我的节点` 的 `select` 代理组，组内列出所有个人节点并保留 `DIRECT`；
- 局域网直连：localhost、`.local`、127/8、10/8、172.16/12、192.168/16、100.64/10；
- 国内直连：`GEOIP,CN,DIRECT` 和 `DOMAIN-SUFFIX,cn,DIRECT`；
- `MATCH,我的节点` 作为最后一条规则。

规则严格按顺序匹配，具体规则在前，通用规则在后。不要添加未明确要求的专用域名规则，不要只保留 `MATCH,我的节点`。

### 4.3 第三方规则

如果用户提供第三方配置，只提取并审查 `rules`：

- 不复制第三方节点、`proxy-providers` 或代理组；
- 将规则中指向第三方代理组的目标改为 `我的节点`；
- 保留国内直连和局域网直连规则；
- 删除重复、无效、无法解析或引用不存在组的规则；
- 将 `MATCH,我的节点` 放到最后；
- 检查 Local profile 中没有第三方节点名、第三方组名或未要求的特殊规则。

禁止修改第三方 Remote profile 原文件。删除第三方 Remote profile 后，`MyServers` Local profile 必须仍可独立使用。

## 5. 部署完整 YAML 订阅

在 VPS 上创建 `~/clash-subscription/`，并完成以下目标：

- 将最新的 `profiles/MyServers.yaml` 同步到 `~/clash-subscription/profiles/MyServers.yaml`，文件权限为 `600`；
- 用高熵随机值生成订阅令牌，令牌文件权限为 `600`；
- 部署轻量 HTTP 服务监听 `0.0.0.0:18080`；
- 只响应精确路径 `/sub/<token>`，其他路径返回 `404`；
- 成功响应状态为 `200`，`Content-Type` 为 `text/yaml`，并设置 `Cache-Control: no-store`；
- 禁用访问日志，避免令牌进入日志；
- 创建用户级 `clash-subscription.service`，启用 linger、开机启动和异常自动重启；
- 确认订阅服务读取的是完整 YAML，而不是 `ss://` 列表或 Base64 文本。

生成并返回给用户的订阅地址格式为：

```text
http://<VPS_IPV4>:18080/sub/<随机令牌>
```

没有域名时必须说明 HTTP 不加密、Base64 不是加密，订阅 URL 本身就是访问凭证；以后应迁移到 Nginx/Caddy + HTTPS。不要在回答中输出 Shadowsocks 明文密码、SSH 密码或私钥。

## 6. 必须实际执行的验收

Codex 必须实际执行并报告结果，不能只回复“已完成”：

1. `sing-box check` 通过，`sing-box-proxy.service` 和 `clash-subscription.service` 均 active/enabled。
2. `40088/tcp`、`40088/udp`、`18080/tcp` 监听正确，防火墙和云安全组状态已确认。
3. 使用 `mihomo -t -f profiles/MyServers.yaml` 检查 Local profile。
4. 请求订阅 URL，确认 HTTP `200`、`text/yaml`、`no-store` 和完整 YAML 字段：`mode`、`proxies`、`proxy-groups`、`rules`。
5. 确认返回内容包含所有个人节点名称、`我的节点` 组、国内/局域网直连规则和最后的 `MATCH,我的节点`。
6. 确认返回内容没有第三方节点、第三方组或未要求的特殊规则。
7. 通过 Mihomo API 检查每个节点的 `alive` 和延迟；API 不可用时标记为未完成，不伪造结果。
8. 通过本地代理端口发起实际 HTTP/HTTPS 请求，并分别验证代理规则和国内直连规则。

所有检查命令都不得打印配置正文或密码。临时响应文件在检查后删除。

## 7. 最终回答格式

```text
SSH：已通过密钥连接 <VPS_IPV4>:<SSH_PORT>
Shadowsocks：<VPS_IPV4>:40088，chacha20-ietf-poly1305，UDP=true
节点：我的节点-01、<其他个人节点>
订阅地址：http://<VPS_IPV4>:18080/sub/<随机令牌>
Local profile：profiles/MyServers.yaml
远端配置：~/clash-subscription/profiles/MyServers.yaml
规则：局域网直连、国内直连、第三方规则已改指向“我的节点”、MATCH 兜底
验证：逐项列出实际结果；无法执行的项目标记为“未完成”
安全提示：HTTP 未加密，订阅地址是访问凭证，请勿公开；有域名后迁移 HTTPS
```

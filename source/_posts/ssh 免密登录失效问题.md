---
title: ssh 免密登录失效问题
date: 2025-2-15 17:57:27
tags:
    - ssh
    - snap
categories: Ubuntu
---
<!--more-->

## [](#问题 "问题")问题

突然间我发现我需要密码才能登录到服务器，但是我仔细对比检查了私钥和秘钥，发现一切正常。这是什么原因呢？

## [](#问题原因 "问题原因")问题原因

原来，前几天我用 snap 安装了 hugo，hugo 需要文件夹权限，为了让 snap 应用访问用户家目录下的文件，我按照网上教程将家目录权限修改为 777。

ssh 为了保证通信安全，防止 key 被篡改或窃取，对目录和文件的权限要求相当严格：

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br></pre></td><td class="code"><pre><code class="hljs bash"><span class="hljs-built_in">chmod</span> 0755 ~<br><span class="hljs-built_in">chmod</span> 700 ~/.ssh<br><span class="hljs-built_in">chmod</span> 600 ~/.ssh/authorized_keys<p>sudo service sshd restart</p></code></pre></td></tr></tbody></table>

SSH 服务程序 sshd 启动时，会 严格检查 home，.ssh 目录和 authorized\_keys 文件的权限。 如果权限不符合要求，sshd 会 拒绝使用公钥认证，并可能退回到密码认证，或者直接拒绝连接，以避免潜在的安全风险。


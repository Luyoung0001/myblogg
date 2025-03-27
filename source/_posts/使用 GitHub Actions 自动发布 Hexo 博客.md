---
title: 使用 GitHub Actions 自动发布 Hexo 博客
date: 2025-2-11 17:57:27
tags:
    - Hexo
    - GitHub_Actions
categories: Blog
---
<!--more-->

## [](#Github-Actions-的原理 "Github Actions 的原理")Github Actions 的原理

按照我的理解就是，首先GitHub 提供了一个虚拟主机，这个主机是干净的。当你需要使用它的时候，你首先得制作一个简单的部署或者测试环境，这是通过 pull docker 镜像或者安装一些软件实现的，甚至你还得配置一些秘钥啥的来访问特定操作（后面有例子）。总之，第一步得配置好环境。

第二步，就是你要在这个机器上要做的事情了，事实上，第一步的时候你已经做了一些事情了，比如安装一些必要的软件来创建所需要的环境。不同的是，这时候就得来做你想要做的事情了。比如编译一些目标、部署一下博客（这篇文章的主题）。

所以 Actions 的原理很好理解，接下来就看要怎么操作才能创建一个 Hexo 生成、部署（github push）的环境了。

## [](#设置权限 "设置权限")设置权限

为什么是设置权限呢？这是因为我们部署的原理的要求。首先我们需要在 A 仓库中写 mark down 博客，然后在根目录中执行一些 hexo 命令来将生成的网页等文件push 到 B 仓库。A 仓库一般是隐私的，存储着我们的 markdown 格式的博客，而 B 仓库一般是我们的目标仓库，一般是公开的，比如 `luyoung0001.github.io`。

问题是，当我们需要将 A 仓库生成的网页等文件 push 到 B 仓库时，是需要权限的，因此我们的环境设置中必须要有权限设置，从而可以使得在 A 目录中通过 `hexo d` 的时候可以成功将生成的代码 push 到 B 仓库中。

权限怎么设置呢？其实 GitHub 已经贴心的将这个需求解决了。我们只需要生成一对密钥，将私钥放在 A，将公钥放在 B。

首先在你本机生成密钥对：

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><code class="hljs bash">ssh-keygen -f github-deploy-key<br></code></pre></td></tr></tbody></table>

一路回车，当前目录下就会生成 `github-deploy-key` 和 `github-deploy-key.pub`。

接着，设置 A 仓库和 B 仓库的公钥和私钥：

对于 A 仓库：进入仓库页面 → Settings → Secrets and variables → actions → New repository secret，Name 填 HEXO\_DEPLOY\_PRI ，Secret 填 github-deploy-key 的内容。

对于 B 仓库：进入仓库页面 → Settings → Deploy keys → Add deploy key，Title 填 HEXO\_DEPLOY\_PUB ，Key 填 github-deploy-key.pub 的内容。

这里的需要注意的是，你需要将私钥整个复制，包括 `-----BEGIN OPENSSH PRIVATE KEY-----`:

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br></pre></td><td class="code"><pre><code class="hljs bash"><span class="hljs-built_in">cat</span> github-deploy-key<br>-----BEGIN OPENSSH PRIVATE KEY-----<br>...<br></code></pre></td></tr></tbody></table>

## [](#Actions-脚本 "Actions 脚本")Actions 脚本

我这里直接给出脚本：

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br><span class="line">25</span><br><span class="line">26</span><br><span class="line">27</span><br><span class="line">28</span><br><span class="line">29</span><br><span class="line">30</span><br><span class="line">31</span><br><span class="line">32</span><br><span class="line">33</span><br><span class="line">34</span><br><span class="line">35</span><br><span class="line">36</span><br><span class="line">37</span><br><span class="line">38</span><br><span class="line">39</span><br><span class="line">40</span><br><span class="line">41</span><br><span class="line">42</span><br></pre></td><td class="code"><pre><code class="hljs YML"><span class="hljs-attr">name:</span> <span class="hljs-string">Deploy</span> <span class="hljs-string">hexo</span> <span class="hljs-string">blog</span><p><span class="hljs-attr">on:</span><br>  <span class="hljs-attr">push:</span><br>    <span class="hljs-attr">branches:</span><br>    <span class="hljs-bullet">-</span> <span class="hljs-string">master</span></p><p><span class="hljs-attr">jobs:</span><br>  <span class="hljs-attr">build:</span><br>    <span class="hljs-attr">runs-on:</span> <span class="hljs-string">ubuntu-latest</span><br>    <span class="hljs-attr">strategy:</span><br>      <span class="hljs-attr">matrix:</span><br>        <span class="hljs-attr">node-version:</span> [<span class="hljs-number">20.</span><span class="hljs-string">x</span>]</p><p>    <span class="hljs-attr">steps:</span><br>      <span class="hljs-bullet">-</span> <span class="hljs-attr">uses:</span> <span class="hljs-string">actions/checkout@v4</span></p><p>      <span class="hljs-bullet">-</span> <span class="hljs-attr">name:</span> <span class="hljs-string">Use</span> <span class="hljs-string">Node.js</span> <span class="hljs-string">${{</span> <span class="hljs-string">matrix.node-version</span> <span class="hljs-string">}}</span><br>        <span class="hljs-attr">uses:</span> <span class="hljs-string">actions/setup-node@v4</span><br>        <span class="hljs-attr">with:</span><br>          <span class="hljs-attr">node-version:</span> <span class="hljs-string">${{</span> <span class="hljs-string">matrix.node-version</span> <span class="hljs-string">}}</span></p><p>      <span class="hljs-bullet">-</span> <span class="hljs-attr">name:</span> <span class="hljs-string">Configuration</span> <span class="hljs-string">environment</span><br>        <span class="hljs-attr">env:</span><br>          <span class="hljs-attr">HEXO_DEPLOY_PRI:</span> <span class="hljs-string">${{secrets.HEXO_DEPLOY_PRI}}</span><br>        <span class="hljs-attr">run:</span> <span class="hljs-string">|</span><br><span class="hljs-string">          sudo timedatectl set-timezone "Asia/Shanghai"</span><br><span class="hljs-string">          mkdir -p ~/.ssh/</span><br><span class="hljs-string">          echo "$HEXO_DEPLOY_PRI" | tr -d '\r' &gt; ~/.ssh/id_rsa</span><br><span class="hljs-string">          chmod 600 ~/.ssh/id_rsa</span><br><span class="hljs-string">          ssh-keyscan github.com &gt;&gt; ~/.ssh/known_hosts</span><br><span class="hljs-string">          git config --global user.name "你的 github 用户名"</span><br><span class="hljs-string">          git config --global user.email "你的 github 邮箱"</span><br><span class="hljs-string"></span>      <span class="hljs-bullet">-</span> <span class="hljs-attr">name:</span> <span class="hljs-string">Install</span> <span class="hljs-string">dependencies</span><br>        <span class="hljs-attr">run:</span> <span class="hljs-string">|</span><br><span class="hljs-string">          npm i -g hexo-cli</span><br><span class="hljs-string">          npm ci</span><br><span class="hljs-string"></span>      <span class="hljs-bullet">-</span> <span class="hljs-attr">name:</span> <span class="hljs-string">Deploy</span> <span class="hljs-string">hexo</span><br>        <span class="hljs-attr">run:</span> <span class="hljs-string">|</span><br><span class="hljs-string">          rm -rf .deploy_git</span><br><span class="hljs-string">          hexo clean</span><br><span class="hljs-string">          hexo d -g</span></p></code></pre></td></tr></tbody></table>

需要注意的是：

+   设置时区很重要。平常我们在自己电脑上部署都是 GMT+8 时区，但是执行 GitHub action 的 runner 在美国，可不是这个时区，所以我们要改下时区，否则如果你的博文地址是 年/月/日 这种形式的话，可能会出现有些博文访问不了的问题。
+   SSH 密钥。这里选择的事 ssh，而之前你部署的时候使用的是 http，那么你需要修改 dev 目录下 \_config.yml 中的 deploy 字段中的 repo，改为 ssh 地址，即：

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br></pre></td><td class="code"><pre><code class="hljs YML"><span class="hljs-attr">deploy:</span><br>  <span class="hljs-attr">type:</span> <span class="hljs-string">git</span><br>  <span class="hljs-attr">repo:</span> <span class="hljs-string">https://github.com/secsilm/secsilm.github.io.git</span><br>  <span class="hljs-attr">branch:</span> <span class="hljs-string">master</span><p><span class="hljs-comment"># 应改为：</span><br><span class="hljs-attr">deploy:</span><br>  <span class="hljs-attr">type:</span> <span class="hljs-string">git</span><br>  <span class="hljs-attr">repo:</span> <span class="hljs-string">git@github.com:secsilm/secsilm.github.io.git</span><br>  <span class="hljs-attr">branch:</span> <span class="hljs-string">master</span></p></code></pre></td></tr></tbody></table>

你也可以配合大模型来仔细查看那个脚本的具体含义，总之就是环境+操作。

## [](#测试 "测试")测试

你可以进行一次更改提交，看看 action 是否正常执行。你可以在 GitHub Actions 页面查看每次运行的日志。

这里一般会遇到几个小问题：

### [](#权限 "权限")权限

可能报这个错误：

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br></pre></td><td class="code"><pre><code class="hljs bash">...<br>Load key <span class="hljs-string">"/home/runner/.ssh/id_rsa"</span>: error <span class="hljs-keyword">in</span> libcrypto<br>git@github.com: Permission denied (publickey).<br>fatal: Could not <span class="hljs-built_in">read</span> from remote repository.<p>Please make sure you have the correct access rights<br>and the repository exists.</p></code></pre></td></tr></tbody></table>

前面已经提到，你必须把私钥所有的内容复制到 repository secret。

### [](#部署失败 "部署失败")部署失败

可能报这个错误：

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br></pre></td><td class="code"><pre><code class="hljs bash">...<br>INFO  540 files generated <span class="hljs-keyword">in</span> 1.98 s<br>INFO  Deploying: git<br>INFO  Clearing .deploy_git folder...<br>INFO  Copying files from public folder...<br>INFO  Copying files from extend <span class="hljs-built_in">dirs</span>...<br>fatal: <span class="hljs-keyword">in</span> unpopulated submodule <span class="hljs-string">'.deploy_git'</span><br>...<br></code></pre></td></tr></tbody></table>

解决办法是：

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><code class="hljs bash"><span class="hljs-built_in">rm</span> -rf .deploy_git<br></code></pre></td></tr></tbody></table>

这个我已经加到脚本了，不会再遇到了。

这篇博客就是通过 Actons 部署，再也不用本地环境了，由于网络的关系，使得某些过程可能会卡很久，但是 Actions 不会存在网络问题，只要你能 push 成功。


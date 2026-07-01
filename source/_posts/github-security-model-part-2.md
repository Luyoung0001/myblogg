---
title: "GitHub 安全机制解析（二）：PAT、GITHUB_TOKEN 与 GitHub App 到底代表谁"
date: 2026-06-29 10:13:13
tags:
  - "GitHub"
  - "GitHub Actions"
  - "PAT"
  - "GitHub App"
categories:
  - "Tooling"
---

## 〇、前言

上一篇主要讲了 GitHub 的权限不是一句“我是 member”能解释的。它至少要看用户、组织、团队、仓库、分支保护、rulesets 等多层关系。

这一篇进入自动化里最容易混乱的部分：token。

在 GitHub 里，我们经常会看到很多种 token：

- Classic PAT；
- fine-grained PAT；
- `GITHUB_TOKEN`；
- GitHub App installation token；
- deploy key；
- OIDC 临时身份。

它们表面上都是一串密钥，放在 secret 里都能被 workflow 读取，发请求的时候也都长得像：

```bash
Authorization: Bearer xxxxxx
```

但它们本质上完全不一样。

我觉得理解这些 token 的关键不是先背它们有什么 scope，而是先问一个问题：

> 这个 token 到底代表谁？

只要这个问题想明白，很多 `403`、Pending approval、跨仓库发布失败就不奇怪了。

这篇继续使用虚构组织：

| 名称 | 类型 | 作用 |
|------|------|------|
| `example-org` | Organization | 示例组织 |
| `example-org/web-client` | Repository | 源码仓库，运行 GitHub Actions |
| `example-org/resource-assets` | Repository | 资源发布仓库，存放 release asset |
| `example-resource-release-bot` | GitHub App | 发布资源的机器身份 |
| `alice` | User | 普通开发者，`web-client` admin |

目标场景很简单：

```text
web-client push main
→ Actions 构建 resource-pack.zip
→ 发布到 resource-assets 的 release
```

问题是：workflow 应该用什么身份去写 `resource-assets`？

## 一、Classic PAT：它代表一个人

Classic PAT 是 Personal Access Token，也就是个人访问令牌。

它的第一性质是：

> Classic PAT 代表某个 GitHub 用户。

比如 `alice` 创建了一个 classic PAT，并把它放到 `example-org/web-client` 的 repository secret 里：

```text
Secret name: RELEASE_PAT
Secret value: ghp_xxxxxxxxx
```

然后 workflow 里这样使用：

```yaml
env:
  GH_TOKEN: ${{ secrets.RELEASE_PAT }}
```

这时 workflow 访问 GitHub API，本质上不是“`web-client` 仓库在访问 GitHub”，而是：

```text
alice 在访问 GitHub
```

也就是说，这个 token 能做什么，取决于：

- `alice` 对目标仓库有什么权限；
- classic PAT 申请了什么 scope；
- 组织是否允许这种 PAT 访问；
- token 有没有过期或被吊销。

Classic PAT 的问题在于 scope 比较粗。比如你为了发布 release，可能给了 `repo` scope。这个 scope 很强，尤其是用户能访问很多私有仓库时，风险就会放大。

![Classic PAT 的权限扩散](/images/mermaid-svg/github-security-model-part-2/classic-pat-scope-spread.svg)

这就是长期 classic PAT 不适合做组织级自动化的原因。它不是一个独立的机器身份，而是把某个人的权限借给了自动化系统。

如果 `alice` 离职、权限变化、token 泄露、账号被锁、2FA 或 SSO 状态变化，都可能影响这个自动化。

一句话总结：

> Classic PAT 能用，但它太像“把人的钥匙交给机器人”。

## 二、Fine-grained PAT：更细，但仍然代表一个人

Fine-grained PAT 相比 classic PAT 有明显改进，它可以限制：

- owner；
- repository；
- permission；
- expiration。

例如 `alice` 创建一个 fine-grained PAT：

```text
Owner: example-org
Repository access: Only selected repositories
Selected repository: resource-assets
Permissions:
  Contents: Read and write
  Metadata: Read
Expiration: 30 days
```

看起来这就很适合发布 release 了。权限范围比 classic PAT 小很多，只能访问 `resource-assets`，而不是 `alice` 能访问的所有仓库。

但它的第一性质仍然是：

> Fine-grained PAT 仍然代表某个用户。

也就是代表 `alice`。

所以当它要访问 organization 资源时，GitHub 还会看组织策略。比如 `example-org` 开启了 PAT 审批策略，那么 `alice` 创建的 fine-grained PAT 可能进入 Pending 状态。

这时 workflow 里会出现一种很典型的失败链路：

```text
web-client Actions
→ 读取 RESOURCE_ASSETS_PAT
→ 用 alice 的 fine-grained PAT 访问 resource-assets
→ GitHub 检查 example-org 是否批准这个 PAT
→ token 状态 Pending
→ 创建 release 失败
→ 403 Forbidden
```

画成图是这样：

![Fine-grained PAT 的组织审批链路](/images/mermaid-svg/github-security-model-part-2/fine-grained-pat-approval.svg)

这里最容易误解的是：`alice` 明明对 `web-client` 是 admin，为什么她的 token 不能用？

因为这个 token 的目标不是 `web-client`，而是 `resource-assets`；并且它访问的是 `example-org` 的组织资源。组织有权决定是否允许个人 token 访问组织仓库。

所以 fine-grained PAT 的安全性比 classic PAT 好，但它仍然有两个天然属性：

- 它绑定个人账号；
- 它可能受组织 PAT 策略和 approval 流程影响。

这也是我们在做长期自动化时要小心的地方。短期、个人、临时脚本，用 fine-grained PAT 很合适；长期、组织级、跨仓库自动化，它未必是最稳的模型。

## 三、GITHUB_TOKEN：当前仓库 Actions 的临时令牌

`GITHUB_TOKEN` 是 GitHub Actions 自动生成的 token。每次 workflow job 运行时，GitHub 都会创建一个临时 token，job 结束后它就失效。

它的第一性质是：

> `GITHUB_TOKEN` 代表当前仓库中的 GitHub Actions 自动化身份。

比如 workflow 运行在：

```text
example-org/web-client
```

那么默认的 `GITHUB_TOKEN` 主要作用范围就是：

```text
example-org/web-client
```

它很适合做当前仓库内的自动化，例如：

- checkout 当前仓库；
- 给当前仓库提交状态；
- 创建当前仓库 release；
- 写当前仓库 issue；
- 上传当前仓库 artifact；
- 读取当前仓库 packages。

用法一般是：

```yaml
permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
```

如果要写当前仓库内容，比如创建当前仓库 release，可以显式给：

```yaml
permissions:
  contents: write
```

这里有一个很重要的安全习惯：

> workflow 里应该显式写 `permissions`，不要依赖默认值。

比如普通构建只需要读仓库：

```yaml
permissions:
  contents: read
```

需要发 issue 才给：

```yaml
permissions:
  issues: write
```

需要发布当前仓库 release 才给：

```yaml
permissions:
  contents: write
```

但是 `GITHUB_TOKEN` 有一个边界：它不是为跨仓库写操作设计的万能钥匙。

回到我们的例子：

```text
workflow 所在仓库: example-org/web-client
目标发布仓库: example-org/resource-assets
```

如果直接用 `web-client` 的 `GITHUB_TOKEN` 去写 `resource-assets`，通常会失败。因为它默认不是 `resource-assets` 的写入身份。

这就像你在 A 房间领到了一张临时门禁卡，这张卡主要开 A 房间。你不能拿它去开 B 房间，然后说“可是 A 和 B 都在同一栋楼里啊”。

## 四、GitHub App：独立机器身份

GitHub App 是另一套模型。

它的第一性质是：

> GitHub App 代表一个独立的机器身份，不代表某个具体用户。

比如我们创建一个 App：

```text
App name: example-resource-release-bot
Owner: example-org
Repository access: only example-org/resource-assets
Permissions:
  Contents: Read and write
  Metadata: Read
```

然后把这个 App 安装到：

```text
example-org/resource-assets
```

这样它就形成了一条很清晰的授权边界：

```text
example-resource-release-bot
只能访问 resource-assets
只能做安装时授予的事情
```

workflow 中不直接保存一个长期 token，而是保存：

- App ID；
- private key；
- installation id，或者通过 API 查询 installation。

运行时 workflow 用 private key 生成一个 JWT，再向 GitHub 换取短期 installation token。

简化链路如下：

![GitHub App installation token 时序](/images/mermaid-svg/github-security-model-part-2/github-app-token-sequence.svg)

这和 PAT 的区别非常大。

PAT 是：

```text
alice 授权 workflow 使用自己的个人权限
```

GitHub App 是：

```text
example-org 安装一个机器身份到指定仓库，并授予指定权限
```

所以 GitHub App 更适合：

- 组织级自动化；
- 长期运行的发布流程；
- 多仓库协作；
- 权限边界清晰的机器人；
- 不希望绑定某个员工账号的场景。

当然，GitHub App 不是没有风险。它的 private key 是高价值凭证。谁拿到 private key，谁就能生成 installation token，代表 App 访问已安装仓库。

所以治理原则是：

- 一个 App 只负责一个职责；
- 只安装到必要仓库；
- 只给必要权限；
- private key 只放到可信源仓库；
- workflow 和 `main` 分支必须受保护；
- 定期轮换 private key。

## 五、四种 token 放在一起比较

现在把它们放在一张表里。

| 类型 | 代表谁 | 典型作用范围 | 优点 | 风险 |
|------|--------|--------------|------|------|
| Classic PAT | 某个用户 | 用户有权限访问的资源，再叠加 scope | 简单、兼容老工具 | scope 粗、长期有效、绑定个人 |
| Fine-grained PAT | 某个用户 | 指定 owner/repo/permission | 权限更细、可设过期 | 仍绑定个人，组织资源可能需要审批 |
| `GITHUB_TOKEN` | 当前仓库 Actions | 当前仓库为主 | 自动生成、短期、易用 | 不适合天然跨仓库写入 |
| GitHub App token | 独立 App 机器身份 | App 安装到的仓库和权限 | 边界清晰、短期、适合组织自动化 | private key 需要严格保护 |

如果只记一句话，我会这样记：

```text
PAT 是个人授权；
GITHUB_TOKEN 是当前仓库自动化授权；
GitHub App 是机器身份安装授权。
```

## 六、案例：为什么 PAT 失败，而 GitHub App 成功

现在回到文章开头的目标：

```text
web-client push main
→ Actions 构建 resource-pack.zip
→ 发布到 resource-assets
```

### 使用 fine-grained PAT 的失败链路

`alice` 在个人设置里创建 fine-grained PAT，选择：

```text
Owner: example-org
Repository: resource-assets
Permission: Contents read/write
```

然后把它放到 `web-client`：

```text
RESOURCE_ASSETS_PAT
```

workflow 读取它之后发布：

```text
web-client Actions
→ 读取 RESOURCE_ASSETS_PAT
→ 用 alice 的 token 访问 resource-assets
→ GitHub 检查 example-org 的 PAT 策略
→ token 还在 Pending
→ API 返回 403
```

这个失败不是因为 `alice` 不会写 workflow，也不是因为 release API 写错了，而是因为授权链路没有闭合。

`alice` 创建了 token，但 `example-org` 还没有批准这个个人 token 访问组织资源。

### 使用 GitHub App 的成功链路

改成 GitHub App 后：

```text
example-resource-release-bot
→ 安装到 example-org/resource-assets
→ 授予 Contents write + Metadata read
```

workflow 中：

```text
web-client Actions
→ 读取 App ID + Private Key
→ 生成 installation token
→ token 代表 example-resource-release-bot
→ App 已安装到 resource-assets
→ App 有 Contents write
→ 发布 release 成功
```

关键差异是：

```text
PAT: 个人授权，受组织 PAT 策略影响
GitHub App: 安装授权，受 App 安装范围和权限影响
```

GitHub App 成功不是绕过安全，而是走了另一套正式授权模型。

## 七、workflow 里应该怎么写权限

即使用了 GitHub App token，也不意味着 `GITHUB_TOKEN` 可以随便开大。

一个比较合理的 workflow 思路是：

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
        run: |
          mkdir -p dist
          echo "example resource pack" > dist/resource-pack.txt
          zip -j dist/resource-pack.zip dist/resource-pack.txt

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

这里有几个点值得注意：

- workflow 自己的 `permissions` 只给 `contents: read`；
- 跨仓库写入不依赖默认 `GITHUB_TOKEN`；
- 发布时使用 GitHub App 生成的短期 token；
- App token 只针对 `resource-assets`；
- private key 放在 `web-client` 的 secrets 里；
- `web-client/main` 必须受保护，因为它能触发发布。

这就是最小权限原则在 Actions 里的实际样子。

## 八、OIDC 又是什么

OIDC 也是 GitHub Actions 里很重要的一种安全机制，但它主要用于让 workflow 获取云厂商或外部系统的临时身份。

比如：

```text
GitHub Actions
→ 通过 OIDC 向云平台证明自己来自 example-org/web-client 的 main 分支
→ 云平台返回短期凭证
→ workflow 使用短期凭证部署服务
```

它解决的问题是：

> 不要把长期云密钥放在 GitHub Secrets 里。

在跨仓库发布 release 这个例子里，我们主要关注 GitHub 内部资源，所以 GitHub App 更直接。如果是部署到 AWS、Azure、GCP、云服务器等外部平台，OIDC 就非常值得优先考虑。

## 九、选择建议

如果只是自己写一个临时脚本，访问自己的仓库：

```text
fine-grained PAT 可以接受
```

如果是当前仓库内的 Actions 自动化：

```text
优先使用 GITHUB_TOKEN
并显式收紧 permissions
```

如果是组织内长期跨仓库自动化：

```text
优先使用 GitHub App
```

如果是访问云平台：

```text
优先考虑 OIDC
```

如果还在用 classic PAT 做长期发布：

```text
应该逐步迁移
```

不是说 classic PAT 完全不能用，而是它在组织自动化里风险太大。它把个人账号、长期凭证、宽权限、自动化流程绑在了一起，一旦出问题，排查和治理都很麻烦。

## 十、结论

token 不是简单的“密码字符串”。在 GitHub 里，每一种 token 背后都有一个身份模型。

Classic PAT 和 fine-grained PAT 代表某个用户；`GITHUB_TOKEN` 代表当前仓库的 Actions 自动化；GitHub App installation token 代表一个被安装到指定仓库的机器身份。

所以排查自动化权限问题时，不要只问：

```text
这个 token 有没有 write 权限？
```

还要问：

```text
这个 token 代表谁？
它想写哪个仓库？
这个身份有没有被目标仓库接受？
它有没有被组织策略拦住？
```

下一篇会把这些机制落到完整 release pipeline 上，重点讲 GitHub Actions 的安全边界、secrets、PR 风险、release asset、sha256 metadata、branch protection 和 rulesets。

## 参考资料

- GitHub Actions 安全加固：[Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- `GITHUB_TOKEN` 权限：[Automatic token authentication](https://docs.github.com/actions/reference/authentication-in-a-workflow)
- Fine-grained PAT approval：[Managing requests for personal access tokens in your organization](https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/managing-requests-for-personal-access-tokens-in-your-organization)
- PAT 策略：[Setting a personal access token policy for your organization](https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/setting-a-personal-access-token-policy-for-your-organization)
- GitHub App 权限：[Choosing permissions for a GitHub App](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/choosing-permissions-for-a-github-app)
- GitHub App 与 OAuth App 区别：[Differences between GitHub Apps and OAuth apps](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/differences-between-github-apps-and-oauth-apps)
- OIDC：[OpenID Connect](https://docs.github.com/en/actions/concepts/security/openid-connect)

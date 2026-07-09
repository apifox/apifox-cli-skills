# Apifox CLI 首次安装引导

本引导用于帮助 AI Agent 或用户完成 Apifox CLI 的首次安装、登录和基础验证。完成后，AI Agent 可通过 Apifox CLI 管理项目资源、运行自动化测试、查询或编辑接口与测试数据。

## 前置要求

- 已安装 Node.js 和 npm/npx。
- 已拥有 Apifox 账号。
- 如需操作指定项目，请确保账号有该项目的访问权限。

## Step 1: 安装或升级 CLI

```bash
npm i -g apifox-cli@latest --registry=https://registry.npmmirror.com/
```

如果国内镜像源下载较慢或安装异常，可切换为 npm 官方源重试：

```bash
npm install -g apifox-cli
```

安装后检查版本：

```bash
apifox --version
```

## Step 2: 获取 API 访问令牌

在 Apifox 客户端或网页端中获取 token：

1. 点击用户头像。
2. 打开「账号设置」。
3. 进入「API 访问令牌」。
4. 创建并复制 token。

安全要求：不要把 token 写入普通日志、聊天记录或仓库文件。

## Step 3: 登录 CLI

```bash
apifox login --with-token <TOKEN>
```

## Step 4: 询问用户记录长期工作项目（推荐）

获取项目 ID：项目 → 项目设置 → 基本设置 → 项目 ID。

可将常用默认 `projectId` 保存到 `.apifox/settings.json`，供 AI Agent 在后续任务中读取本文件：

```json
{
  "projectId": 123456
}
```

当用户需求没有明确指定项目时，请优先使用这里保存的 `projectId`。

建议同时确保 `.apifox/.gitignore` 包含：

```gitignore
*.private.*
```

## Step 5: 验证安装和登录状态

查看可用命令：

```bash
apifox --help
```

查看登录身份：

```bash
apifox whoami
```

查看项目列表：

```bash
apifox project list
```

如果已准备项目 ID，可进一步验证资源查询命令，例如：

```bash
apifox environment list --project <PROJECT_ID>
```

## Step 6: 安装 AI Agent Skill（!强制必需）

安装 Apifox AI Agent Skills，让 AI Agent 知道如何正确使用 Apifox CLI：

```bash
npx -y skills add https://apifox.com
```

该命令会进入交互流程，可按需选择要安装的 Skills、目标 AI Agent 和安装范围。

或通过 GitHub 仓库安装：

```bash
npx -y skills add https://github.com/apifox/apifox-cli-skills
```

如果当前环境无法访问官方安装源，请按所用 AI Agent 的规则手动安装 Apifox AI Agent Skills。调用 `WebFetch` 获取 https://apifox.com/.well-known/agent-skills/index.json，并根据其中的 `url` 下载和安装需要的 `SKILL.md` 文件；推荐安装全部 8 份，至少安装 `apifox-cli`。

## 常见问题

| 现象 | 处理 |
|------|------|
| `apifox: command not found` | 重新执行全局安装，并确认 npm global bin 在 PATH 中 |
| 提示未登录 | 重新执行 `apifox login --with-token <TOKEN>` |
| 项目不存在或无权限 | 确认项目 ID 正确，并确认当前 token 对应账号有项目权限 |
| 命令参数不确定 | 执行 `apifox <command> --help` 查看最新用法 |

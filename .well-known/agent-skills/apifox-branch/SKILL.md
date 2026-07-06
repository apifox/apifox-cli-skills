---
name: apifox-branch
description: Apifox 分支协作：普通分支、迭代分支、AI 分支、pick-to、merge、merge-request、保护分支、AI 写入权限与分支资源变更流程。用户要在分支上修改资源、创建 AI 分支或合并改动时使用。
metadata:
  requires:
    bins: ["apifox"]
  cliHelp: "apifox branch --help; apifox merge-request --help"
---

# 分支协作与 AI 写入

> 前置条件：先阅读 `../apifox-cli/SKILL.md`。若旧总入口与本 skill 的领域规则冲突，以当前 CLI help 和本 skill 为准。

具体命令参数以当前 CLI help 为准。本 skill 只补充分支协作中最容易误用的规则：分支上下文、AI 分支、pick-to、写入权限、保护分支和合并确认。Agent 在分支上写入后必须用同一 `--branch` 回读和验证，不要跨分支用默认视角判断资源是否存在。

## 何时使用

- 用户指定 `--branch` 或说在某个分支上操作。
- 创建、查看、更新、归档、删除分支。
- 创建 AI 分支并修改源分支资源。
- 将源分支资源 pick-to 到 AI 分支。
- 合并分支或创建/审批 merge request。
- 遇到 AI 写入权限限制。

## 命令入口

- 分支管理：`apifox branch --help`
- 合并请求：`apifox merge-request --help`
- 资源命令通常都支持 `--branch <branchName>`。查询、创建、更新、运行测试时保持同一个分支上下文。

## 先判断怎么改

分支写入前先问清楚用户要哪种方式：

- 直接编辑目标分支：适合用户明确允许直接修改，且目标分支没有保护或权限限制。
- 通过 AI 分支编辑：适合 AI 代改接口、测试用例、测试场景或配置；也适合目标分支受保护、未开启外部 AI 直接写入，或用户希望先隔离验证再合并。

开始写入前必须确认：

- 项目 ID。
- 源分支和目标分支。
- 要修改的资源范围。
- 是否允许后续创建 merge request 或直接合并。

## AI 分支流程

AI 分支用于隔离 AI 修改，避免直接污染源分支。

```text
确认源分支和目标变更
  -> 创建 AI 分支
  -> pick-to 已有资源
  -> 在 AI 分支修改
  -> 验证 get/run/report
  -> preview merge-request
  -> 用户确认后 create merge-request 或 merge
```

标准步骤：

1. 创建 AI 分支时指定来源：`--from <sourceBranchName>`。
2. 使用规范名称：`ai/年月日-from-来源分支名-功能或模块名`，例如 `ai/20260312-from-main-userRegister`。
3. 修改源分支已有资源前，先用 `branch pick-to` 导入到 AI 分支；新建资源不需要 pick-to。
4. 在 AI 分支上修改、查询、验证时都带 `--branch <aiBranchName>`。
5. 按资源类型验证结果，例如 `get`、`run`、本地报告或云端报告检查。
6. 合并前先 `merge-request preview`，让用户确认影响范围。
7. 用户确认后，再 `merge-request create` 或 `branch merge`；目标主分支受保护时优先 merge request。

AI 分支注意点：

- AI 分支初始为空，不会自动复制源分支全部资源。
- `branch get` 可能返回 `type: SPRINT` 且 `isAiBranch: true`；这不代表操作时应使用 `--type sprint`。AI 分支相关操作仍按当前 help 使用 `--type ai`。

## 分支参数规则

- 优先使用分支名：`--branch <branchName>`。
- 兼容纯数字 branchId，但分支名更直观，日志和排障更清楚。
- 同一任务内查询、创建、更新、运行测试必须带同一个分支上下文，避免在 main/default 视角找不到资源。

## pick-to 规则

- 必须带 `--type ai`。
- 只支持从主分支或普通迭代分支导入到 AI 分支。
- 不能导入到主分支。
- 不能以 AI 分支作为来源分支。
- 使用 `--include-endpoint-cases` 可在导入接口时同时导入接口用例。

## 写入权限规则

- 主分支、迭代分支、通用分支可能要求开启“外部 AI 编辑权限”。
- 典型错误信息包含 `Automation caller branch required`。
- 创建 AI 分支可能成功，但写入 AI 分支内容仍可能因“允许 AI 修改 AI 分支内容”关闭而失败。
- 遇到写入受限，不要自动切策略；让用户选择开启直接编辑权限，或改走 AI 分支。
- 目标主分支受保护时，优先 `merge-request preview/create`，不要直接 `branch merge`。

## 不可违反规则

1. 不要在用户未确认时创建 AI 分支。
2. 不要在 AI 分支里修改源分支已有资源而不先 pick-to。
3. 不要在分支任务中省略 `--branch <branchName>`。
4. 不要在用户未确认时 merge、create merge-request 或审批 merge request。
5. 不要删除、归档分支，除非用户明确要求。
6. 遇到 `Automation caller branch required` 时，必须停下来问用户选择权限策略，不要自动创建或切换 AI 分支。

## 常见恢复

| 现象 | 处理 |
|------|------|
| get 不带 branch 找不到资源 | 带 `--branch <branchName>` 重试 |
| AI 分支里找不到源资源 | 先 `branch pick-to` 导入 |
| `projectBranchState` 有计数但 AI 分支页面为空 | 不要误判为已导入；AI 分支仍需 pick-to |
| run-config 在分支下 404 | 确认 case/endpoint/environment/branch 均存在后转 `apifox-cli-checkup` |
| merge 被保护分支拦截 | 改走 `merge-request` |

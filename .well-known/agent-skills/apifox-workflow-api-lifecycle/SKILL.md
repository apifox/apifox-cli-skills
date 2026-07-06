---
name: apifox-workflow-api-lifecycle
description: Apifox API 生命周期工作流：从需求到接口设计、Schema、环境、Mock、测试用例、文档导出/发布和分支合并。用户要“创建一组 API 并补齐测试/Mock/文档”时使用。
metadata:
  requires:
    bins: ["apifox"]
---

# API 生命周期工作流

> 开始前必须读取 `../apifox-cli/SKILL.md`。若旧总入口与本 workflow 的领域规则冲突，以当前 CLI help 和本 workflow 为准。按步骤再读取 `apifox-branch`、`apifox-import-export`、`apifox-test-case`；涉及复杂测试流程时读取 `apifox-test-scenario`，涉及执行和 CI 时读取 `apifox-test-automation`。

具体 endpoint、schema、environment、test-report 命令以当前 CLI help 为准。本流程用于从需求到 API 资产、测试、报告和分支合并的端到端交付，重点关注资源顺序、质量门禁和交付验证。Agent 执行时优先形成 `help/schema -> get -> validate -> write -> run/report` 闭环，不要仅凭旧文档或记忆写入。

## 适用场景

- 根据需求创建或更新一组接口。
- 导入 OpenAPI/Postman 后整理接口、Schema、目录和测试。
- 从代码库、PRD、需求文档、测试文档或讨论内容生成 API spec 后导入 Apifox。
- 给接口补 Mock、测试用例、文档和发布设置。
- 在 AI 分支完成 API 变更并准备合并。

## 工作流

```text
确认项目/分支
  -> 如需从代码/文档导入，先做 spec 生成和质量门禁
  -> 设计接口和 Schema
  -> 配置环境和变量
  -> 配置 Mock
  -> 创建接口测试用例
  -> 运行测试并查看报告
  -> 导出/发布文档
  -> 合并或创建 MR
```

## Step 1：确认上下文

按当前 CLI help 确认身份、项目、目标分支和是否需要 AI 分支。

- 用户要直接改主分支/迭代分支时，确认 AI 写入权限或走 AI 分支。
- 如果通过 AI 分支改已有资源，先用 `branch pick-to` 导入源资源。

## Step 2：设计 API 资源

按当前 CLI help 使用 endpoint、schema、response-component、security-scheme、folder 等 API 设计资源命令。

最小边界：endpoint 是接口定义，test-case 是接口下测试用例，不要混写；schema、response-component、security-scheme 等可复用资源应先创建，再由 endpoint 引用；环境变量不要写进 common-parameter。

如果任务是从代码库、PRD 或文档生成并导入 API spec，先读取 `../apifox-import-export/SKILL.md`，完成生成器搜索、OpenAPI 指标校验、tags 分组和临时项目导入验证，再进入接口设计或测试补充。

- 先建 `schema`、`response-component`、`security-scheme` 等可复用资源，再引用到 endpoint。
- 创建后必须 `endpoint get` 验证真实保存结构。

## Step 3：配置环境

按当前 CLI help 使用 environment、variables、database-connection、vault 等环境和运行上下文命令。

- 没有合适环境时创建或更新环境。
- 敏感变量不要出现在最终回复中。
- 运行测试和 CI 交付命令建议显式带 `--environment`，避免默认环境变化导致不可复现。

## Step 4：补测试用例

读取 `../apifox-test-case/SKILL.md`。

- 不要创建空壳 case。
- 创建后 `test-case get` 验证步骤、断言、提取器被保存。

## Step 5：验证和交付

- 测试报告异常先区分本地报告、云端上传和下载概要；必要时转 `apifox-cli-checkup`。
- 前端展示异常转 `apifox-cli-checkup`。
- 需要合并时转 `apifox-branch`。
- 如果新建 API 的测试运行返回 404 或断言失败，先检查 Mock/环境是否配置，不要直接判定 endpoint 或 test-case 创建失败。Mock 未配置导致运行失败是执行环境问题，不等于设计资产未保存。

## 不可违反规则

1. 不要跳过 project/branch 确认。
2. 不要直接在受保护主分支写入。
3. 不要只创建接口不验证保存结构。
4. 不要只创建空测试用例。
5. 不要把测试失败当作接口创建失败；分别定位 API 定义、环境、测试用例和执行报告。

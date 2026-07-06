---
name: apifox-test-automation
description: Apifox 自动化测试执行、套件与 CI：test-suite、scheduled-task、runner、apifox run、执行参数、迭代数据、报告上传和 CI 回归。复杂 test-scenario 步骤建模请使用 apifox-test-scenario。
metadata:
  requires:
    bins: ["apifox"]
  cliHelp: "apifox test-scenario --help; apifox test-suite --help; apifox run --help"
---

# 自动化测试执行、套件与 CI

> 前置条件：先阅读 `../apifox-cli/SKILL.md`。若旧总入口与本 skill 的领域规则冲突，以当前 CLI help 和本 skill 为准。涉及复杂场景步骤建模时读取 `../apifox-test-scenario/SKILL.md`，涉及接口 case 时读取 `../apifox-test-case/SKILL.md`。环境、变量、报告相关命令以当前 CLI help 为准；下面重点说明执行边界和常见坑。

具体命令参数以当前 CLI help 为准。执行前重点确认资源边界、空套件风险、runner/定时任务约束、CI 报告边界和运行后排查顺序。Agent 场景下把 run 结果作为验收动作；CI 场景下以退出码、报告文件和上传状态作为门禁依据。

## 何时使用

- 创建、更新、运行测试套件。
- 运行已有测试场景。
- 配置定时任务或 CI 回归。
- 管理 runner 或检查 runner 状态。
- 需要 `apifox run` 的执行参数、reporters、迭代、变量覆盖、SSL、超时、报告上传。

## 资源边界

| 用户诉求 | 优先资源 |
|----------|----------|
| 单接口下的测试用例 | `test-case`，转 `apifox-test-case` |
| 多步骤业务流程建模 | `test-scenario`，转 `apifox-test-scenario` |
| 多场景集合回归 | `test-suite` |
| 定时执行 | `scheduled-task` |
| 私有执行机 | `runner` |
| 查看执行结果 | `test-report`，按当前 CLI help 查询或下载报告 |

## 命令入口

使用当前 CLI help 查询 `test-suite`、`scheduled-task`、`runner` 和 `run` 的参数。`run --help` 覆盖 reporters、out-dir、upload-report、iteration、变量覆盖、SSL、超时和 `--carry-runtime-variables` 等参数。

`test-suite create --name` 会创建空套件，客户端可展示但 `items: []`。除非用户明确要占位套件，否则“创建回归套件/自动化套件”必须通过 `--file` 或后续 update 加入 items，并 `get` 验证 `items` 非空。

非空套件应使用 `cli-schema get test-suite-create` 中的前端兼容结构，例如 `STATIC_TEST_CASE` + `testCases[].id` 引用已有测试用例。不要使用 legacy shorthand，例如 `{ testScenarioId }` 这类会被 schema validate 故意拦截的写法。

Runner 是团队级执行资源，创建前必须确认团队和用途。当前实际常用组合是 `runnerType=GENERAL`、`serverType=SELF_HOSTED`，不要把 runner 当成项目内轻量资源随手创建。

## 创建/更新规则

复杂测试场景创建/更新请使用 `apifox-test-scenario`。

定时任务创建不要给空壳示例。虽然 schema required 可能只标 `name/cronExpression/runOn`，真实可用任务通常还需要有效 runner、`TEST_SUITE` entityId 等上下文；`runOn` 仅限当前 CLI help/schema 支持值，例如 `APP/TSHGR/OSHGR`，不要写未支持的 `CLOUD`。

更新前必须先 `get` 原始结构，避免覆盖步骤、变量、场景引用或套件成员。

## 运行参数

常见运行参数以 `apifox run --help` 和具体 run 命令 help 为准。CI 场景重点确认 environment、reporters、out-dir、upload-report、iteration、变量覆盖、超时、bigint 和是否需要 `--carry-runtime-variables`。

建议 CI 最小命令形态：

```bash
apifox test-suite run <suiteId> --project <projectId> --environment <environmentId> --reporters cli,json,junit --upload-report
```

## 执行后动作

- 本地报告：检查 `--out-dir` 和 `--out-file`。
- 云端报告：仅在运行时带 `--upload-report` 后，按当前 CLI help 执行 `test-report list/get/download`。
- 失败排查：先看 CLI 输出、JSON report、agentHints，再定位到具体 scenario/suite/case。
- 不带 `--upload-report` 时，云端 `test-report list` 不会出现本次本地执行结果。
- CI 中建议显式指定 `--environment`，token 使用 CI secret 注入，不要写入仓库。

## 常见恢复

| 现象 | 处理 |
|------|------|
| 创建场景后步骤不对 | 转 `apifox-test-scenario`；确认 create 后是否 update steps |
| 套件运行为空 | `test-suite get` 确认包含的场景/用例 |
| 套件 `items: []` | 这是空占位套件，不是有效回归套件 |
| CI 找不到环境 | 按当前 CLI help 使用 `environment list/get` 确认 environmentId |
| runner 不可用 | `runner check`，再看 runner get/list |
| 报告没有步骤详情 | 先区分本地 JSON、云端上传、下载接口概要，再必要时转 `apifox-cli-checkup` |

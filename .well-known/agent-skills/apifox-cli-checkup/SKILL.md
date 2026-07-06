---
name: apifox-cli-checkup
description: Apifox CLI 使用检查与版本确认：命令成功但页面没看到、创建后 list/get 找不到、测试运行失败、报告缺失、agentHints/help/实际行为不一致，或怀疑本机 CLI 版本不是最新时使用。
metadata:
  requires:
    bins: ["apifox"]
  cliHelp: "apifox --help; apifox update --help; apifox cli-schema --help"
---

# CLI 使用检查与版本确认

> 前置条件：先阅读 `../apifox-cli/SKILL.md`。若旧总入口与本 skill 的规则冲突，以当前 `apifox <command> --help` 和本 skill 为准。根据资源类型再读取对应业务 skill。

本 skill 用于公开 CLI 使用排查，不依赖内部接口或内部代码。目标是确认命令、项目、分支、环境、资源 ID、报告位置和 CLI 版本是否一致，再回到具体业务 skill 修正资源结构。Agent 排查时优先以当前 help、schema validate、get 回读和 agentHints 组成闭环，不要只看 summary 文案。

## 何时使用

- CLI 返回成功，但 Apifox 页面没看到资源或展示不完整。
- 创建后 `list/get` 找不到资源。
- 测试用例、测试场景或测试套件运行失败。
- 本地报告或云端报告找不到、没有步骤详情。
- `agentHints`、help、示例或实际命令行为互相矛盾。
- 用户或测试同学反馈“我这边没有这个参数”“命令不认识”，需要确认 CLI 版本。

## 先做 5 件事

1. 记录原始命令、projectId、branch、resourceId、environmentId、文件路径和是否带 `--api-base-url`。
2. 执行 `apifox --version`，确认本机 CLI 版本；参数不存在时先看是否需要更新。
3. 执行对应 `apifox <command> --help`，以当前公开 help 为准，不凭旧文档或记忆使用参数。
4. 用对应 `list/get` 回读资源，确认是否写入了预期项目、分支、模块、目录或分类。
5. 如果是运行或报告问题，先区分本地报告和云端报告：未带 `--upload-report` 时，不要去云端报告列表找本次结果。

## 版本检查

先看版本和命令 help：

```bash
apifox --version
apifox import --help
apifox test-case category --help
```

如果测试说明里要求的新参数没有出现在 help 中，优先更新 CLI：

```bash
apifox update
```

非交互环境或确认直接更新时：

```bash
apifox update --yes
```

如果自动更新提示影响排查，可以让用户在 shell 配置里设置禁用每日检查，但这不会影响手动 `apifox update`：

```bash
export APIFOX_CLI_DISABLE_UPDATE_CHECK=1
```

版本排查结论必须写清：当前 `apifox --version`、命令路径（如 `which apifox`）、缺失的参数名、建议更新方式。

## help 与提示冲突

- 公开文档、agentHints、历史示例和实际行为不一致时，优先以当前 `apifox <command> --help` 和实测为准。
- 不主动推荐 help 未公开的隐藏别名。
- `success=false` 时以真实 `success` 字段和退出码为准，不要相信 summary 里的成功语义。
- 如果命令提示下一步但实际参数不存在，记录为 CLI 提示问题，并使用 help 中公开的替代命令。

## 页面看不到资源

优先检查：

- 是否写入了正确 project。
- 是否带了正确 `--branch`。
- 是否资源在 AI 分支中，且页面当前查看的是同一分支。
- 是否写入到了预期模块或目录。
- 测试用例是否使用了有效 `categoryId`。
- Apifox 原生格式二次导入时，是否因为模块策略导致资源进入了新模块。

常用回读命令：

```bash
apifox endpoint list --project <projectId> --branch <branchName>
apifox test-case list --project <projectId> --endpoint <endpointId> --branch <branchName>
apifox test-scenario get <scenarioId> --project <projectId> --branch <branchName> --with-case-detail
```

如果 `get/list` 能看到，但页面看不到，先确认页面筛选条件、分支、模块、目录、分类是否一致；不要直接重建资源。

## 测试用例排查

- 创建前必须用 `apifox test-case category --project <projectId>` 获取有效 `categoryId`。
- 当前 `test-case category` 不支持 `--endpoint`；按接口查看用例用 `test-case list --endpoint <endpointId>`。
- `test-case get` 能看到结构，只说明资源已保存，不代表 requestBody、断言、提取变量和脚本一定能运行。
- 运行失败时检查 environment、变量、请求体、前后置脚本、断言和报告详情。

## 测试场景排查

- `test-scenario create` 只创建场景元数据；复杂步骤需要后续 `import-steps`、`add-ref` 或 `update --file`。
- 创建或更新后先 `test-scenario get --with-case-detail`，确认步骤树和 HTTP 详情展开正常。
- 步骤间变量为空时，检查是否运行完整场景、步骤编号、响应路径、提取变量和环境选择。
- 不要把 `test-case` 的结构直接写成 `test-scenario` 步骤。

## 运行与报告排查

- 未指定 `--environment` 时，服务端可能使用项目默认环境；为了复现，建议显式指定。
- 本地报告看 `--out-dir` 和 `--out-file`。
- 只有运行时带 `--upload-report`，云端 `test-report list/get/download` 才能看到本次报告。
- 报告没有步骤详情时，先对比本地 JSON 和云端报告；本地也没有详情时，回查运行对象和资源结构。
- 有副作用的测试不要在生产环境默认执行。

## 常见分流

| 现象 | 处理 |
|------|------|
| 新参数不识别 | `apifox --version`、`which apifox`，必要时 `apifox update --yes` |
| 创建成功但页面没看到 | 检查 project、branch、模块、目录、分类、页面筛选 |
| test-case 页面看不到 | 检查 `categoryId`、endpoint、branch；用 `test-case list --endpoint` 回读 |
| 场景步骤不展示 | `test-scenario get --with-case-detail`，确认 create 后是否真正写入 steps |
| run-config 或运行前失败 | 确认 case/scenario/endpoint/environment/branch 都存在且一致 |
| 云端报告找不到 | 确认运行时是否带 `--upload-report` |
| agentHints 和 help 冲突 | 以当前 help 和实测为准，记录提示问题 |

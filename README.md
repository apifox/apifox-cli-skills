# Apifox CLI Skills

Apifox CLI 的 AI Agent Skills 用于帮助 AI Agent 更准确地使用 Apifox CLI 管理项目资源、设计接口、编排测试、运行自动化测试和处理分支协作。

## 安装

推荐使用 `skills` CLI 安装：

```bash
npx -y skills add https://github.com/apifox/apifox-cli-skills
```

该命令会进入交互流程，可按需选择要安装的 Skills、目标 AI Agent 和安装范围。

也可以从 Apifox 官方站点安装：

```bash
npx -y skills add https://apifox.com
```

安装后可查看已安装的 skills：

```bash
npx -y skills list
```

## Skills 列表

| Skill | 作用 |
| --- | --- |
| `apifox-cli` | Apifox CLI 总入口，适合项目资源管理、通用命令查询和未细分到具体业务域的 CLI 任务。 |
| `apifox-cli-checkup` | Apifox CLI 使用检查与版本确认，用于排查命令成功但页面不可见、资源找不到、报告缺失、help 或 agentHints 不一致等问题。 |
| `apifox-branch` | Apifox 分支协作，包括普通分支、迭代分支、AI 分支、pick-to、merge、merge-request、保护分支和 AI 写入权限。 |
| `apifox-import-export` | Apifox 导入导出，包括 OpenAPI、Postman、Swagger、Apifox 原生格式导入导出和导入质量检查。 |
| `apifox-test-case` | Apifox 接口测试用例，包括 test-case、test-data、测试步骤、断言、提取变量、处理器、分类和 direct run。 |
| `apifox-test-scenario` | Apifox 测试场景建模，包括导入接口/用例/其它场景步骤、场景引用、条件、循环、等待、脚本、数据库和变量调试。 |
| `apifox-test-automation` | Apifox 自动化测试执行、套件与 CI，包括 test-suite、scheduled-task、runner、`apifox run`、执行参数和报告上传。 |
| `apifox-workflow-api-lifecycle` | Apifox API 生命周期工作流，从需求到接口设计、Schema、环境、Mock、测试用例、文档导出/发布和分支合并。 |

## 使用建议

- 遇到 Apifox CLI 任务时，优先使用 `apifox-cli`。
- 涉及具体业务域时，再加载对应的专项 skill，例如测试场景用 `apifox-test-scenario`，分支协作用 `apifox-branch`。
- 执行写入类操作前，优先查看当前 CLI help 和 JSON Schema，并使用 `cli-schema validate` 校验 payload。
- 如果 skill 内容和当前 CLI 输出不一致，以当前 CLI 输出为准。

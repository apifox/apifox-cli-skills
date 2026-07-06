---
name: apifox-import-export
description: Apifox 导入导出与质量门禁：导入 OpenAPI/Postman/Apifox 原生格式，导出 OpenAPI/HTML/Markdown/Postman/Apifox 原生格式；导入前校验 spec 完整性、tags 分组、schema/body 覆盖率，导入后验证计数、模块和资源可见性。
metadata:
  requires:
    bins: ["apifox"]
  cliHelp: "apifox import --help; apifox export --help"
---

# 导入导出与质量门禁

> 前置条件：先阅读 `../apifox-cli/SKILL.md`。若旧总入口与本 skill 的领域规则冲突，以当前 CLI help 和本 skill 为准。

具体命令参数以当前 CLI help 为准。导入前重点检查文件格式、OpenAPI 质量门禁、路由骨架风险、tags 分组、原生格式模块策略、ignoreCount 和是否需要干净项目验证。Agent 导入后必须按 agentHints、list/get 和必要的 run/report 做结果确认，不要只依据导入命令返回成功判断完成。

## 何时使用

- 从代码库、PRD、需求文档、测试文档、讨论记录生成 API spec 并导入 Apifox。
- 导入 OpenAPI、Postman、HAR、Apifox 原生格式等项目数据。
- 配置自动导入。
- 导出 OpenAPI、Markdown、HTML、Postman 或 Apifox 原生格式。
- 需要判断生成的 spec 是否只是“路由骨架”。
- 迁移、备份或复制 Apifox 项目，尤其是需要控制模块复用或新建时。

## 不应使用

- 把已有 endpoint、test-case 或 test-scenario step 导入/复制到测试场景：转 `apifox-test-scenario`，使用 `test-scenario import-steps` 或 `test-scenario add-ref`。
- 精细维护测试场景步骤、变量、断言或处理器：转 `apifox-test-scenario`。

## 命令入口

使用当前 CLI help 查询 `import`、`import auto-import`、`export` 和 `export settings` 的参数。导入前不要跳过下面的质量门禁。

## 核心原则

1. 优先查项目内已有生成器，不要先手写提取脚本。
2. 不要把“路径完整”误判为“接口 spec 完整”。
3. 导入前必须输出质量指标。
4. 完整性和可读性要同时验收。
5. 导入策略不确定时，优先新建临时/修正版项目验证，不要污染已有项目。
6. 导入结果里的大量 `ignoreCount` 是风险信号，不是普通成功。
7. Apifox 目录分组依赖 OpenAPI operation tags。
8. 不要假设文件扩展名代表实际格式。

## 标准流程

### Step 0. 明确任务类型

先判断用户要做的是哪一类任务：

| 任务 | 下一步 |
|------|------|
| 从代码库、PRD、文档生成 spec 并导入 | 继续 Step 1 |
| 直接导入已有文件 | 从 Step 3 开始 |
| 配置自动导入 | 先读取当前 CLI help 和 schema，再创建配置 |
| 导出文档或 OpenAPI | 直接使用 export 命令，导出后检查文件是否生成 |
| 迁移或备份 Apifox 项目 | 使用 Apifox 原生格式导出导入，并确认模块导入策略 |
| 把已有资源导入测试场景 | 转 `apifox-test-scenario`，不要走 OpenAPI import |

如果用户没有指定目标项目、团队、导入策略或是否允许创建新项目，先根据上下文判断；不确定时只问一个最小必要问题。

### Step 1. 搜索项目内已有生成器

遇到“从源码/文档/PRD/测试文档/讨论生成 API spec / 文档 / 类型”的需求，先搜索项目内是否已有：

```text
openapi
swagger
routegen
route-gen
docs generator
api docs
schema generator
cmd/openapi
cmd/*openapi*
```

正确方向：

- 先运行项目自带生成器。
- 优先使用能抽取 handler request/response struct、DTO、schema 的工具。
- 不要先手写脚本从 router 文件提取路径，否则很容易只得到 method + path 骨架。

如果找不到生成器，再考虑从框架路由、注释、DTO、schema、测试或文档生成 spec；生成后仍必须继续 Step 3 之后的质量门禁。

### Step 2. 生成 spec 并确认文件格式

- 运行项目自带生成器或官方生成命令。
- 保存原始产物，不要直接覆盖用户已有文件。
- 不要根据扩展名判断格式。
- 读取生成文件前，先看文件开头或用支持 JSON/YAML 的 parser。
- 生成器可能输出 YAML 到 `.json` 文件；如果实际是 YAML，应改名为 `.yaml` 或按 YAML 解析。

### Step 3. 统计导入前质量指标

导入前必须实际解析生成的 OpenAPI 文件，并报告真实统计值。不要使用示例值、默认值或占位值。

必须统计：

| 指标 | 含义 | 用途 |
|------|------|------|
| `paths` | OpenAPI paths 数量 | 判断接口规模 |
| `operations` | 实际 operation 数量 | 判断导入规模 |
| `schemas` | `components.schemas` 数量 | 判断模型完整度 |
| `writes` | POST/PUT/PATCH 等写接口数量 | 判断 body 覆盖目标 |
| `withBody` | 写接口中有 requestBody 的数量 | 判断 requestBody 覆盖率 |
| `emptyObjectBodies` | requestBody schema 是空对象的数量 | 判断是否路由骨架 |

输出时必须写“真实统计结果”，例如使用表格或 JSON 均可，但每个值必须来自当前文件解析结果。

### Step 4. 判断 spec 完整性

- 不按固定 schema 数量判定质量。
- 小型、纯 GET、健康检查、webhook 透传、无 JSON body 项目，`schemas=0/1` 可以接受。
- 大项目如果接口很多、写接口很多，但 schemas 极少，视为疑似路由骨架 spec。
- 大量 POST/PUT/PATCH 的 body 缺失或是空 `{}`，是强风险信号。
- 只有 method + path 不能证明 requestBody、response、schema 完整。

风险判断：

| 现象 | 判断 | 处理 |
|------|------|------|
| 接口规模大、写接口多，但 schemas 极少 | 疑似路由骨架 spec | 继续查 DTO、handler request/response struct、项目内生成器或文档来源 |
| 写接口很多，但 withBody 覆盖明显不足 | requestBody 不完整 | 继续补充 request DTO 或换生成器 |
| emptyObjectBodies 很多 | 强风险，可能只是空壳 body | 不要作为最终 spec 导入 |
| 小型、纯 GET、健康检查、webhook 透传 | schemas 很少可能合理 | 结合业务形态判断，不强制失败 |

如果判定为路由骨架 spec，不要导入为最终版本；最多导入临时项目用于探索。

### Step 5. 校验 tags 和文档可读性

API spec 不只要模型完整，还要在 Apifox 中可读、可导航、分组合理。

必须检查：

- operation 是否有业务化 `tags`。
- tags 是否按业务域分组，而不是按 URL path 机械分组。
- schema 是否保留完整。
- operationId、summary、description 是否可读。

推荐 tags 形态：

```yaml
paths:
  /api/v1/<resource>:
    post:
      tags:
        - <业务域名称>
```

不推荐导入后目录按技术路径机械展开，例如：

```text
api / v1 / <resource>
```

更推荐按产品模块、业务域、资源域或用户可理解的功能域设置 tags，让 Apifox 客户端中的目录服务于业务导航，而不是服务于代码路径还原。

如果生成器产物的 tags 明显过粗或机械来自技术路径，例如大量接口都落在 `api`、`v1`、`rest`、`service`、`controller` 等目录下，必须停在导入前，先询问用户是否允许生成一个 tags 更合理、项目文档更整洁的修正版 spec。用户同意前，不要把该 spec 导入为最终项目。

### Step 6. 选择导入项目

- 导入策略不确定时，优先创建临时/修正版项目验证。
- 不要用新 spec 反复导入污染已有项目。
- 如果已有项目里导入过骨架 spec，再导入完整 spec 可能出现旧接口 ignore、新接口追加、项目混杂。
- 临时项目命名建议表达版本和目的，例如 `API - Full Spec`、`API - Grouped Full Spec`。

### Step 7. 执行导入并检查结果

导入后不要只看命令成功，还要看结果计数。

如果导入结果中 `ignoreCount` 明显偏高，不能把“命令执行成功”当成“导入质量合格”。必须继续判断：

- 是否导入到了已有项目。
- 是否接口匹配策略不符合预期。
- 是否旧 spec 和新 spec 混杂。
- 是否应该改用干净项目重新导入。

### Step 7A. Apifox 原生格式导入策略

Apifox 原生格式适合项目迁移、备份、跨项目复制和局部资源搬迁。常用命令：

```bash
apifox import --project <projectId> --format apifox --file ./project.apifox.json
```

默认模块策略是 `match-name`：源模块名与目标项目模块名唯一匹配时导入已有模块，未匹配时新建模块。二次导入同一个项目时，优先使用默认策略，避免重复创建同名模块。

```bash
apifox import --project <projectId> --format apifox --file ./project.apifox.json --module-import-mode match-name
```

如果测试目标是每次复制一套全新模块，使用 `new`：

```bash
apifox import --project <projectId> --format apifox --file ./project.apifox.json --module-import-mode new
```

目标项目存在多个同名模块，或需要精确控制源模块去向时，使用 `--module-map`。它可以重复传，且优先级高于 `--module-import-mode`：

```bash
apifox import --project <projectId> --format apifox --file ./project.apifox.json \
  --module-map "商店 API=8049476" \
  --module-map "管理 API=8049482"
```

常用映射写法：

| 写法 | 用途 |
|------|------|
| `"源模块名=目标模块ID"` | 导入到指定已有模块 |
| `"source:源模块ID=目标模块ID"` | 源模块重名时用源模块 ID 精确指定 |
| `"源模块名=default"` | 导入到目标项目默认模块 |
| `"源模块名=new"` | 只让该源模块新建 |

只指定部分源模块也可以；未指定的源模块继续按 `--module-import-mode` 处理。目标模块建议使用模块 ID，避免目标项目同名模块歧义。

导入后必须验证：

- 模块数量是否符合预期，二次导入不应无意新增同名模块。
- API、Schema、测试用例、测试场景、WebSocket、Socket.IO 等资源数量是否符合导入结果。
- 单接口测试用例分类是否可见。
- 如导入测试套件，抽查套件里的场景和用例引用是否指向新项目资源。

### Step 7B. Apifox 原生格式导出策略

全量备份或迁移：

```bash
apifox export --project <projectId> --format apifox --output ./project.apifox.json
```

只导出指定接口或目录，适合局部搬迁或最小复现：

```bash
apifox export --project <projectId> --format apifox --scope apis --api-ids 1001,1002 --output ./selected.apifox.json
apifox export --project <projectId> --format apifox --scope folders --folder-ids 2001 --output ./folder.apifox.json
```

按标签导出，适合按业务域交付：

```bash
apifox export --project <projectId> --format apifox --scope tags --include-tags pet,store --exclude-tags deprecated --output ./tagged.apifox.json
```

默认会包含接口调试和单接口用例；如果只需要接口定义，显式关闭：

```bash
apifox export --project <projectId> --format apifox --no-include-api-cases --output ./apis-only.apifox.json
```

导出后检查文件存在、大小合理，并在临时项目中做一次导入验证，尤其是发给他人或用于迁移前。

### Step 8. 回读验证并汇报

- 回读接口列表，确认接口总数与导入结果一致。
- 抽查至少一个读接口和一个写接口；写接口要确认 requestBody、response、schema 引用正常。
- 如导入了数据模型，抽查 schema 能否正常回读。
- 如导入 Apifox 原生格式，额外确认模块策略、测试用例分类、测试场景和测试套件引用。
- 最终汇报必须包含文件路径、质量指标或导出范围、导入项目、导入计数、抽查结果和遗留风险。

## 不可违反规则

1. 不要先手写路由提取脚本，先查项目内生成器。
2. 不要把接口数量多等同于 spec 完整。
3. 不要跳过导入前质量指标。
4. 不要在已有项目上反复试错导入。
5. 不要忽略大量 `ignoreCount`。
6. 不要导入 tags 混乱、无法按业务导航的 spec 作为最终成果。
7. 不要把 schema 极少的大项目 spec 当成最终版本。

## 常见恢复

| 现象 | 处理 |
|------|------|
| 导入成功但只有路径 | 回查生成器是否只抽路由，继续找 DTO/handler schema |
| schemas 极少 | 结合 operations/writes/withBody/emptyObjectBodies 判断是否路由骨架 |
| 大量空 body | 回查 request DTO 或项目生成器 |
| Apifox 目录按 URL 分组 | 重写 operation tags 后重新导入到干净项目 |
| 大量 ignoreCount | 判断是否污染已有项目，必要时新建干净项目验证 |
| JSON.parse 失败 | 检查实际是否 YAML，不要信扩展名 |
| 原生格式二次导入后出现重复模块 | 检查是否使用默认 `match-name`，目标同名模块是否唯一；必要时用 `--module-map` |
| 目标项目有多个同名模块 | 不要让 CLI 自动猜，使用 `--module-map "源模块名=目标模块ID"` |

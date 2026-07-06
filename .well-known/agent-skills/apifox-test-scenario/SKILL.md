---
name: apifox-test-scenario
description: Apifox 测试场景建模：test-scenario 的查询、创建、更新、删除和运行；导入接口、单接口用例或其它场景步骤；添加场景引用步骤；复杂步骤编排、接口步骤/条件/循环/等待/脚本/数据库等步骤衔接、前后置操作、变量引用、断言、提取器和场景调试最佳实践。用户要创建或维护复杂自动化测试流程时使用。
metadata:
  requires:
    bins: ["apifox"]
  cliHelp: "apifox test-scenario --help"
---

# 测试场景建模

> 前置条件：先阅读 `../apifox-cli/SKILL.md`。若旧总入口与本 skill 的领域规则冲突，以当前 CLI help 和本 skill 为准。涉及单接口 case 时读取 `../apifox-test-case/SKILL.md`，涉及执行/CI/报告边界时读取 `../apifox-test-automation/SKILL.md`。环境和变量命令以当前 CLI help 为准。

具体命令参数以当前 CLI help 为准。创建和更新测试场景时重点处理步骤保存语义、导入/引用边界、变量传递、复杂步骤字段风险和运行验证边界。Agent 维护场景时优先使用导入/引用命令和 `get --with-case-detail` 回读真实结构，再做局部 update。

## 何时使用

- 创建或更新测试场景。
- 设计多步骤自动化流程，例如登录 -> 创建资源 -> 查询 -> 断言 -> 清理。
- 编排多种步骤类型、条件分支、循环、等待、脚本、数据库操作或外部程序。
- 处理前置/后置操作、变量引用、变量提取、断言链路。
- 排查场景步骤衔接失败、变量为空、步骤顺序错误、场景创建后前端展示异常。

## 不应使用

- 单个接口下的测试用例：转 `apifox-test-case`。
- 只运行已有场景、套件或 CI 命令：转 `apifox-test-automation`。
- 只查看报告：按当前 CLI help 使用 `test-report`；执行和报告边界参考 `apifox-test-automation`。
- 只配置环境、变量、数据库连接：按当前 CLI help 使用 environment、variables、database-connection 等命令。

## 核心边界

| 概念 | CLI 资源 | 说明 |
|------|----------|------|
| 接口测试用例 | `test-case` | 绑定某个 endpoint 的 case，适合单接口验证 |
| 测试场景 | `test-scenario` | 多步骤流程编排，支持复杂步骤衔接 |
| 测试套件 | `test-suite` | 场景/用例集合与回归组织 |
| 环境 | `environment` | 提供 baseUrl、变量、服务配置 |
| 测试数据 | `test-data` | 场景或 case 的迭代数据来源 |

## 命令入口

使用当前 CLI help 查询 `test-scenario` 的参数。`import-steps` 和 `add-ref` 未必出现在顶层“可用命令”列表，但各自支持 `--help`，可用于导入步骤和添加场景引用。

关键事实：`test-scenario create` 只保存元数据。即使 create payload 里包含 `steps`，步骤也不会在 create 阶段保存。正确流程是先 create 元数据，再 `get --with-case-detail`，再用 `import-steps`、`add-ref` 或 `test-scenario update --file` 添加或修改 steps。

常见导入优先使用高层命令，不要手写复杂 HTTP 绑定结构。已验证的入口包括：

```bash
apifox test-scenario import-steps <scenarioId> --project <projectId> --source endpoint --ids <endpointIds> --sync manual
apifox test-scenario import-steps <scenarioId> --project <projectId> --source test-case --endpoint <endpointId> --ids <testCaseIds> --sync manual
apifox test-scenario import-steps <scenarioId> --project <projectId> --source test-scenario --from-scenario <sourceScenarioId> --step-ids <stepIds>
apifox test-scenario add-ref <scenarioId> --project <projectId> --scenario <sourceScenarioId>
```

语义边界：

- `import-steps` 是复制/导入为当前场景自己的步骤。
- `add-ref` 是添加一个引用其它场景的步骤，不复制源场景内部步骤。
- `--sync manual` 是默认模式，适合导入后补业务参数、变量引用和断言。
- `--sync auto` 只适用于 endpoint/test-case 来源，表示随源接口定义或单接口用例自动同步；`test-scenario` 来源不支持 auto。
- CLI 是非交互式命令，不会缺 ID 时自动列出全量资源；缺 endpoint、case、scenario 或 step ID 时，跟随 agentHints 先 list/get 定位。

简化创建必须给完整必填参数，但这只会创建空场景。除非用户明确要占位场景，否则“创建自动化测试/场景”不能停留在空场景。

## 场景建模标准流程

1. 明确业务目标：验证什么流程、成功条件是什么、失败如何清理。
2. 确认 project、branch、environment。
3. 列出涉及的 endpoint、case、环境变量、测试数据、数据库连接或脚本。
4. 如果在 AI 分支或迭代分支创建场景，只调整场景本身时不需要先 pick 被引用接口或用例；只有要修改被引用接口、单接口用例或源场景本身时，才从主分支或对应来源迭代分支 pick 所需资源到当前分支。
5. 如果已有相似场景，先 `test-scenario list/get` 读取作为模板。
6. 用自然语言先设计步骤图，再转 JSON，不要边猜字段边写。
7. 获取 `test-scenario-create` schema 创建元数据。
8. 创建后 `test-scenario get --with-case-detail` 回读完整结构。
9. 导入已有资源时优先用 `import-steps` 或 `add-ref`，不要直接手写复杂绑定字段。
10. 导入后再次 `test-scenario get --with-case-detail`，确认步骤写入且 HTTP case/detail 展开正常。
11. 导入 API definition 或 test-case 后，检查 params、headers、body、脚本变量是否只是 schema 示例值；需要业务值时再用 `test-scenario update --file` 补齐。
12. 精细编辑时获取 `test-scenario-update` schema，基于完整结构添加或修改 steps。
13. `cli-schema validate` 通过后 update，再 `test-scenario get --with-case-detail` 确认 `steps` 非空且结构正确。

`test-scenario-update` schema 已包含步骤、前后置处理器、断言、提取变量和枚举值说明。首次编写 processor 时先看 schema，不要凭经验猜字段名、枚举值或旧格式。

## 步骤设计最佳实践

设计每个步骤时都写清：

```text
stepName: 这一步做什么
stepType: 使用哪类步骤
input: 请求、脚本、SQL、等待条件或引用变量
output: 提取哪些变量
dependsOn: 依赖哪些上游步骤输出
assertions: 成功条件
onError: 失败后继续、停止或清理
cleanup: 是否需要后置清理
```

复杂流程建议分层：

```text
准备数据
鉴权/登录
主流程操作
结果查询与断言
副作用校验
清理资源
```

## 数据传递和变量引用

- 创建前列出数据来源：环境变量、全局变量、前置步骤请求/响应、迭代数据、脚本输出。
- 步骤间传递数据优先考虑 Apifox 原生“读取前置步骤的运行结果”，例如 `{{$.1.response.body.token}}`、`{{$.2.response.body.data.id}}`；它只在自动化测试场景中生效，需要运行完整场景，单独运行某个步骤无法取值。
- 同一数据需要多次引用、跨模块复用，或希望命名更稳定时，使用后置操作的提取变量，再用 `{{token}}` 等变量引用。
- 随机后缀、临时标识符、跨步骤生成但不来自响应的数据，适合写入环境变量，例如 `pm.environment.set('runSuffix', suffix);`。
- 在脚本中使用前置步骤结果时，不要直接写 `{{...}}`；使用 `pm.variables.get("$.1.response.body.token")`。
- 步骤引用依赖步骤 ID/number，插入、删除、重排步骤后必须同步检查 `{{$.步骤号...}}` 是否仍指向正确步骤。
- 不要假设响应路径一定是 `body.data.id`；先根据真实响应确认路径，例如 `{{$.2.response.body.id}}`、`{{$.2.response.body.data.id}}`、`{{$.2.response.body.data[0].id}}`。
- 生成 payload 时必须原样保留 `{{...}}` 占位符，不要转义、拆分或改写，否则运行时无法替换。
- 变量为空时，优先检查是否运行完整场景、步骤 ID、JSONPath、响应结构、步骤执行顺序和环境选择。
- 这些步骤间传递、`{{$.步骤号...}}` 和 `forEach` 规则只适用于 `test-scenario`，不要写入单接口 `test-case`。

列表响应可作为 `forEach.parameters.array`，循环内用当前元素引用后续字段：

```json
{
  "type": "forEach",
  "parameters": {
    "array": "{{$.1.response.body}}",
    "disableOnError": false
  }
}
```

循环子步骤中引用当前元素：`{{$.2.element.id}}`。如果默认取数组第一个元素，要显式写出索引，例如 `{{$.7.response.body.data[0].id}}`；如果业务要求特定元素，先筛选，不要假设列表顺序稳定。

## 步骤内容和处理器结构

Apifox CLI 不会自动格式化场景内容，写入什么字符串，客户端就展示什么字符串。写入前先预格式化字符串字段：

- 接口 Body：`requestBody.data`。
- 脚本内容：`parameters.data`。
- 前置/后置脚本：`preProcessors[*].data`、`postProcessors[*].data`。

规则：

- JSON Body 仍然写成字符串，不要写成对象。
- 多行内容用 `\n` 写入。
- 预格式化只影响客户端展示可读性，不改变执行语义。
- 处理器使用扁平结构 `{ id, type, data, defaultEnable, enable }`，不要写旧式嵌套 `{ type, config }`。
- 处理器建议带稳定 `id`，尤其是 `assertion`、`extractor`、`customScript`；缺少 `id` 可能 validate 通过但运行器或客户端解析异常。
- 提取全局变量时，`data.variableType` 使用 `globals`；如需指定生效范围，`data.shareScope` 优先使用 `PROJECT`。`TEAM` 是团队范围，可能依赖增值能力，除非用户明确要求团队范围，否则不要默认使用。
- 写入前照常跑 `cli-schema validate`。
- 通过 `import-steps` 从 API definition 导入的步骤可能只有结构或 schema 示例值；运行前必须回读并按业务场景补齐 params、headers、body、脚本变量等。

示例：

```json
{
  "requestBody": {
    "type": "application/json",
    "data": "{\n  \"name\": \"Demo\",\n  \"description\": \"Readable in client\"\n}"
  },
  "postProcessors": [
    {
      "id": "postProcessors.0.customScript",
      "type": "customScript",
      "data": "pm.test('返回 ID', function () {\n  var body = pm.response.json();\n  pm.expect(body.data.id).to.exist;\n});",
      "defaultEnable": true,
      "enable": true
    },
    {
      "id": "postProcessors.1.extractor",
      "type": "extractor",
      "data": {
        "variableName": "project_pet_name",
        "variableType": "globals",
        "shareScope": "PROJECT",
        "subject": "responseJson",
        "expression": "$.name"
      },
      "defaultEnable": true,
      "enable": true
    }
  ]
}
```

## 断言和脚本规则

常规校验优先用可视化 `assertion`，自定义脚本只作为兜底能力。

断言规则：

- 引用接口形成 HTTP 步骤时，优先使用原接口自带的契约测试/响应校验；默认契约通常已覆盖成功状态码，例如 200。
- 已启用接口契约测试时，不要重复添加“HTTP 状态码为 200”这类同义断言；只补充契约外的业务断言。
- 只有没有合适契约校验，或需要验证额外业务字段时，才单独添加 `assertion`。
- HTTP 状态码、JSON 字段存在/相等、文本包含等常规校验优先用可视化 `assertion`，不要默认写 `customScript`。
- 可视化断言字段使用当前 schema 枚举：HTTP 状态码用 `httpCode`，不要用 `responseCode`；JSON 字段用 `responseJson`，不要用 `responseBody`；全文包含用 `responseText` + `include`；比较符用 `equal`，不要用 `equals`。

```json
{
  "type": "assertion",
  "data": {
    "name": "返回 ID",
    "subject": "responseJson",
    "comparison": "exists",
    "path": "$.data.id",
    "value": ""
  },
  "defaultEnable": true,
  "enable": true
}
```

脚本规则：

- 自定义脚本只用于可视化断言覆盖不了的逻辑，例如复杂数组筛选、条件判断、二次请求、XML 转换或 schema 校验。
- Apifox 脚本运行时通过 `pm` 对象读写变量、访问响应和定义断言；脚本断言使用 `pm.test(...)` 包裹。
- 不要在 `pm.test` 外裸调用 `pm.response.json()`，避免空响应或非 JSON 响应导致错误难定位。
- 不要凭经验扩展运行上下文字段。

```js
pm.environment.set("variable_key", "variable_value");
pm.variables.set("variable_key", "variable_value");

pm.test("Status code is 200", function () {
  pm.response.to.have.status(200);
});

pm.test("JSON value equals expected", function () {
  var jsonData = pm.response.json();
  pm.expect(jsonData.value).to.eql(100);
});
```

## 前后置和清理规则

- 前置操作适合鉴权、准备数据、生成随机值、初始化数据库状态。
- 后置操作适合清理测试数据、撤销副作用、释放资源。
- 有副作用的场景必须考虑清理步骤，避免污染环境。
- 数据库、外部程序、写入型接口步骤属于高风险操作，执行前确认环境不是生产环境。

## 常见复杂步骤处理

| 步骤类型 | 建模建议 |
|----------|----------|
| 接口请求步骤 | 先确认 endpoint/case 或直接请求结构，输出关键响应字段 |
| 条件分支 | 条件表达式必须基于已存在变量或响应字段 |
| 循环/迭代 | 明确最大次数、退出条件和失败策略 |
| 等待/轮询 | 明确等待上限，避免无限等待 |
| 脚本步骤 | 输入输出变量要显式，避免隐式全局副作用 |
| 数据库步骤 | 先确认连接、SQL 只操作测试数据，敏感信息不输出 |
| 清理步骤 | 即使主流程失败，也尽量能执行清理 |

## 字段风险提醒

- 步骤顺序看 `number`，不要只看回读数组顺序。
- HTTP 步骤必须带正确接口绑定信息，避免变成孤立请求或客户端展示异常。可用 `type: "http"` + `bindId: <endpointId>` + `bindType: "API"` + `syncMode: "SYNC_WITH_API"` + `httpApiCase.apiDetailId` 导入接口；后端会生成 `relatedId`。
- 常见 endpoint/test-case/test-scenario 导入不要手写上述复杂绑定结构，优先用 `test-scenario import-steps`；引用其它场景优先用 `test-scenario add-ref`。
- 复杂业务场景通常需要 `syncMode: "MANUAL"` 承载定制参数、变量引用和断言，仅绑定 API 不等于可用业务场景。
- 容器步骤 `group/if/else/loop/forEach/onError` 必须包含 `disable=false`、`parameters`、`isOpen=true`、`children=[]`。
- 子步骤放在 `children`，不要平铺。
- `group` 展示名写在 `parameters.name`，不是顶层 `name`。
- `if` 和条件 `break` 使用 `parameters.keyVariable` + `operator` + `valueVariable`，不要写 `expression`。
- `else` 和 `onError` 的 `parameters` 应为空对象 `{}`。
- `loop` 使用 `parameters.count`，不是 `times` 或其他字段。
- `delay` 使用 `parameters.timeout`，单位毫秒，不是 `duration`。
- `script` 步骤使用 `parameters.type="customScript"`、`parameters.data=<JS code>`、`enable=true`，不要写 `language/code`。
- `customHttp` 的 URL 字段叫 `customHttpRequest.path`，不是 `url`，且需要完整请求字段。
- `testCaseRef` 字段是 `relatedId`，不是 `caseId`。
- `relatedId` 是后端为场景步骤生成的 HTTP case ID，新建步骤时不要从旧场景复制复用。
- `test-scenario get --with-case-detail` 用于确认步骤树和 HTTP case/detail 内部配置，例如 Body、Header、postProcessors；如果用户明确要求验证可运行性，再执行运行验证并检查报告。
- `onError` 目前存在 CLI/schema/get 成功但客户端展开和配置展示不稳定的风险；涉及 onError 时不能仅凭 validate/get 成功判断可用，需要客户端确认。

## 验证和调试规则

- `cli-schema validate` 只保证基础 JSON 结构，不保证 runner、客户端或处理器一定能正确解析。
- 更新或导入后先 `test-scenario get --with-case-detail` 确认步骤树非空且结构正确，但不要把 get 成功当成可运行。
- 创建、更新或导入步骤后默认先回读确认，不要自动运行；只有用户明确要求运行、调试失败或交付前需要验证可运行性时，才执行 `test-scenario run` 并检查报告。
- 运行失败时看报告步骤详情，区分失败发生在请求、前置脚本、后置脚本、断言、变量引用还是环境。
- 调试复杂场景时不要反复覆盖同一个业务场景；必要时新建版本化场景，避免旧步骤和新结构混杂。
- 临时删除脚本/断言可以用于定位问题，但不能把“删除功能”当成最终修复。

## 不可违反规则

1. 不要把 `test-case` 的结构直接当作 `test-scenario` 步骤结构；导入单接口用例到场景优先使用 `test-scenario import-steps --source test-case --endpoint <endpointId> --ids <caseIds>`。
2. 不要只创建一个空场景名；必须包含可展示、可运行的步骤结构。
3. 不要误以为 create 能保存 steps；必须 create 元数据后再用 `import-steps`、`add-ref` 或 `update --file` 写入 steps。
4. 不要凭经验猜复杂步骤字段；先读 schema 和现有场景模板。
5. 不要让后续步骤引用未确认来源的数据；先确认它来自步骤响应、环境变量、迭代数据、extractor 或脚本输出。
6. 不要在不确认环境的情况下执行有副作用步骤。
7. 更新场景前必须先 `get --with-case-detail` 原结构，避免覆盖整个步骤树。
8. 不要反复覆盖同一个复杂业务场景做调试；必要时新建版本化场景，避免旧步骤和新结构混杂。

## 调试流程

| 现象 | 处理 |
|------|------|
| 场景创建成功但前端步骤不展示 | `test-scenario get` 看真实保存结构，必要时转 `apifox-cli-checkup` |
| 后续步骤变量为空 | 检查上游 extractor、响应路径、变量名和执行顺序 |
| 场景 run 失败但单接口成功 | 检查步骤间变量传递、环境、前置脚本和依赖顺序 |
| 循环或等待卡住 | 检查退出条件、最大次数、timeout |
| 清理没执行 | 检查失败策略和后置步骤配置 |
| 报告没有步骤详情 | 先按 `apifox-test-automation` 区分本地/云端报告，再必要时转 `apifox-cli-checkup` |

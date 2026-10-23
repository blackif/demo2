# readme 生成规范（程序包式样书）

本文件是 AI 生成 `document/readme.md` 与 `document/readme.json` 时必须遵守的规范。

## 1. 目的与范围

- 目的：根据一个 SAP 程序包内的全部文件，按模板生成式样书（程序式样 + 测试式样）。
- 不做：程序逻辑是否正确的检查（由其他 skill 负责）；不修改源码；只依据程序包内的文件，不连接 SAP 系统。
- 只写入两个文件：`<程序包>/document/readme.json` 和 `<程序包>/document/readme.md`。

## 2. 输入与输出

| 项目 | 内容 |
|---|---|
| 输入 | 程序包路径（如 `sap/zpack/zfidemo_0001`）；可选：输出语言（默认 `ja`） |
| 模板 | `sap/document/template/readme_ja.md` |
| Schema | `sap/document/template/readme.schema.json` |
| 输出 | `<程序包>/document/readme.json`、`<程序包>/document/readme.md` |

调用示例：「按 sap/document/skills/readme_generation.md，为 sap/zpack/zfidemo_0001 生成 readme」。

## 3. 基本原则

1. **不编造。** 只写能从程序包内文件确认的内容。
2. **以标题为粒度，要么 confirmed，要么 unknown，没有"一知半解"。**
   - 能判断 → `confirmed`，由 AI 写入 md。代码里确实没有该项内容时，content 为空数组，md 输出"-"。
   - 不能判断 → `unknown`，content 为 `null`，md 输出 `- [ ] 要確認`，由人后续补写。
   - 区别：**"-" 表示 AI 已判断且代码中没有；"要確認" 表示 AI 无法判断。**
3. **readme.json 对人只读，人只编辑 md。** 不要求人修改 json。
4. 不记录源码位置（行号等）。
5. 类型（TYPES 等）、数据元素、域不列入任何清单。
6. 常量、变量等的业务含义不凭名称断定；需要时写成「（コード上の名称：xxx）」。

## 4. 生成流程

1. **判断是否首次**：`document/readme.json` 不存在即为首次执行。若首次执行时 `document/readme.md` 已存在，该 md 会被整体覆盖，覆盖前须向用户提示。
2. **遍历文件**：遍历程序包内除 `document/` 以外的所有文件，不论目录名、扩展名、拼写是否规范。逐个完整阅读，按内容判断对象类型，登记到 `files`。无法判断类型或无法理解的文件：`objectType` 为 `unclassified`，`status` 为 `unknown`，`role` 为 `null`。
   - 如果 Issue「要求」里附有「程序包文件清单」，以它为起点，按路径逐个读取；同时尽量用目录树核对，二者不一致时以 `main` 上的目录树为准，并在报告中说明。
   - 枚举文件必须用**目录列表或文件树**，**不得用代码搜索**（搜索有索引延迟，也搜不到没有扩展名的文件）。有终端时用 `ls -A`；用 API 时用 `GET /repos/{owner}/{repo}/contents/{程序包路径}?ref=main` 逐级列出子目录，或 `GET /repos/{owner}/{repo}/git/trees/main?recursive=1` 后按路径前缀筛选。只能读单个文件的接口（如连接器的 `fetch_file`）不能对目录路径调用，会报错。不要猜测目录名，一切以枚举结果或 Issue 清单为准。
   - 读取每个文件的完整内容用 `contents` API（带 `ref=main`）或 blob 的 `sha`，不得以搜索片段代替。
   - 枚举完成后核对：登记到 `files` 的数量等于列表中 `document/` 以外的文件数。
3. **逐项分析**：理解整个程序后，对 16 个标题逐项判断 confirmed 或 unknown，按第 5 节填写 `content`。非首次执行时，status 的继承按第 7 节处理。
4. **写 readme.json 并校验**：用 `readme.schema.json` 校验，通过后才能继续；不通过则修正。
5. **非首次执行**：按第 7.2 节做人工改动检测。
6. **渲染 md**：按第 8 节渲染，写入 `document/readme.md`。
7. **自检并报告**：按第 9 节。

## 5. 各项目的填写规则

`fields` 中的 field 值是模板里的标题原文。`content` 的键见第二列。

| 标题 | content 键 | 写什么 | 判为 unknown 的情形 |
|---|---|---|---|
| `## 業務要件` | businessRequirement | 业务目的、背景、课题、期待结果（段落） | 包内文件（注释等）没有明确写出时；不得从处理内容反推业务目的 |
| `## 機能要件` | functionalRequirement | 依次写 入力／出力／処理条件／機能上の制約，各一段，均取自代码可确认的行为 | 无法确定入口和输入输出时 |
| `## 業務フロー` | businessFlow | Mermaid sequenceDiagram，参与者只有 ユーザ（U）和 システム（S） | 包内文件无法确认用户操作与启动契机时（如入口 FM 没有调用方） |
| `## 処理概要` | processSummary | 启动条件、做什么、实现什么的整体说明 | 无法概括整体处理时 |
| `### 1. 機能概要` | functionOverview | 機能名（取自程序注释，英文注释保留原文并可附日译；无注释则用对象名）、処理方式、処理概要 | 无法概括时 |
| `### 2. 処理詳細` | processSteps | 按源码实际执行顺序；第一步写入口调用链（如 FM → Include → 类）；终止和跳过的条件写进对应步骤 | 无法梳理执行顺序时 |
| `### 3. 処理条件` | processConditions | 所有决定是否执行、如何分支的判断，每行 判定対象／判定条件／成立時／不成立時；No. 渲染时自动编号 | 代码里没有任何判断时是空数组，不是 unknown |
| `### 4. 処理概要図` | flowchart | Mermaid flowchart TD，严格按实际控制流；不添加源码中不存在的处理和分支；节点标签用双引号包起来，不使用裸的 < > | 无法梳理控制流时 |
| `### **前提条件：**` | prerequisites | 处理成立所需的设定、输入数据、扩展设定、执行条件；每条以类别名开头（如「設定情報：」）；没有依据的不写 | 基本不会 unknown |
| `### **制約条件：**` | constraints | 处理对象／对象外、适用范围、因数据或设定状态产生的限制、数据不足时的动作；每条以类别名开头 | 基本不会 unknown |
| `## 依存関係` | dependencies | 见 6.2 | 种类判断不出时（见 6.3） |
| `### 使用公開API` | publicApis | 见 6.2 | 同上 |
| `### **メッセージ**` | messages | MESSAGE 语句、带消息的 RAISE EXCEPTION、消息类中的消息；正常的条件不成立不当作错误；没有则为空数组 | 无法判断时 |
| `### データモデル・項目情報` | dataModel | 程序读写的主要数据源、结构、字段的用途和处理内容，处理内容末尾括注（参照）（更新）等；同一对象的多个字段可合并为一行 | 无法判断时 |
| `### 例外処理` | exceptionHandling | label 固定 5 项，顺序固定：データ取得結果が0件の場合／設定情報が取得できない場合／対象条件を満たさない場合／例外発生時／その他。代码中没有对应处理的，text 写 "-" | 无法判断时 |
| `# テスト仕様書` | testCases | 从代码分支推导 正常系・異常系・条件分岐・境界；只写条件，不写具体数值；majorNo 为连续整数，minorNo 为「大-小」形式（如 1-1） | 无法判断时 |

`# README`、`# 要件書`、`# 概要書`、`## 前提/制約条件`、`## 補足情報` 是结构标题，不是字段，不进 `fields`。

## 6. 判定规则

### 6.1 对象类型

按文件内容判断（如 `CLASS ... DEFINITION`、`FUNCTION ...`、`extend type ... with ...`、`INCLUDE`），不依赖目录名和扩展名。

### 6.2 使用公開API 与 依存関係的分工（互不重复）

| 放入 | 内容 |
|---|---|
| 使用公開API | 程序用到的表、视图、CDS 视图；自制、标准均算；包内定义的也算 |
| 依存関係 | 其余的依赖对象：类、FM、Include、DDIC 结构（含追加结构）、异常类等；包内、包外均可 |

只列实际调用、引用、读写的对象，不凭推测添加。

### 6.3 种类（objectType / apiType）的判断

- 包内有定义的，以定义为准。
- 包外的对象，依据 SAP 标准知识和命名规则（如 I_、C_、ZI_、ZC_ 开头为 CDS 视图）判断。
- 判断不出时，该项（依存関係或使用公開API）整体 unknown。

### 6.4 语言

默认日语（`meta.language`）。指定其他语言时，content 中的文字用该语言，标题、表头、例外処理的固定 label 仍保持模板原文。对象名、字段名用大写。

## 7. 再生成（非首次执行）

### 7.1 status 的继承与迁移

| 旧 json 的 status | 本次 AI 的判断 | 处理 |
|---|---|---|
| 不存在（首次） | confirmed 或 unknown | 按判断写入 json 和 md |
| confirmed | confirmed | 先做 7.2 检测，再用新内容覆盖 md 对应区块 |
| confirmed | unknown | status 改为 unknown，md 对应区块改为 `- [ ] 要確認`（先做 7.2 检测） |
| unknown | unknown | md 对应区块原样保留（不论仍是"要確認"还是人已补写） |
| unknown | AI 现在可以判断 | **status 保持 unknown**，md 对应区块原样保留，并在报告中提示"该项 AI 现在可以判断" |

unknown 项的 content 为 `null`。人补写的内容不会随代码变化自动更新，由人自行维护。

### 7.2 人工改动检测

用旧 readme.json（旧的 status 和 content）按第 8 节规则重新渲染，取其中 confirmed 项的区块，与现有 md 中同标题的区块比较。

- 不一致：视为人改过 AI 管理的区块。在覆盖前向用户列出这些标题和差异摘要，然后继续覆盖，不等待回复。
- 旧 json 的 `schemaVersion` 与当前 schema 不同时，检测结果不可靠，向用户说明。

### 7.3 区块的界定

区块 = 标题的下一行起，到下一个任意标题（`#` 至 `###`）之前。围栏代码块内以 `#` 开头的行不算标题。有子标题的字段，其区块只含到第一个子标题之前的部分，子标题各自是独立区块。

### 7.4 找不到标题

现有 md 中找不到某个字段的标题（被改名或删除）时，该区块按本次结果重新生成，并在报告中说明。

## 8. md 渲染规则

### 8.1 总则

- 以 `readme_ja.md` 为准：标题层级、顺序、标题文字、表头都照抄。
- 删除模板中全部 HTML 注释（含多行），不得残留 `{{`。
- 结构标题（`# README` 等 5 个）只输出标题，不放内容。
- 块与块之间空一行，文件以换行结尾。

### 8.2 confirmed 的渲染

| content 键 | 渲染方式 |
|---|---|
| businessRequirement、functionalRequirement、processSummary | 每个元素一段，段间空一行 |
| prerequisites、constraints | 每个元素一条 `- ` |
| functionOverview | `- **機能名：** …`、`- **処理方式：** …`、`- **処理概要：** …` 三条 |
| processSteps | `1. **title**`，各 detail 以 3 个空格缩进的 `- ` 列出 |
| exceptionHandling | `- **label：** text` |
| businessFlow、flowchart | ` ```mermaid ` 围栏代码块 |
| processConditions | 表格，No. 从 1 自动编号 |
| dependencies、publicApis、messages、dataModel、testCases | 表格 |

表格的表头与分隔行照抄模板；单元格内的 `|` 写成 `\|`，不使用换行。

### 8.3 空数组（confirmed，代码中没有）

- 表格：表头 + 一行，每个单元格都是 `-`（processConditions 的 No. 也是 `-`）。
- 其他：一行 `\-`。

### 8.4 unknown 的渲染

- 标题下的正文写一行 `- [ ] 要確認`，其后空一行。人补写内容后会删除这一行。
- 表格类字段：`- [ ] 要確認` 写在表格上方；之后空一行，再输出表头、分隔行和一个空白行（各单元格留空，不写文字）。
- 文字、列表、Mermaid 类字段：只写 `- [ ] 要確認`，不输出代码块或表格。
- `reason` 只留在 json 中，不写入 md。

## 9. 自检与报告

### 自检清单（全部满足才算完成）

1. readme.json 通过 schema 校验。
2. `fields` 覆盖 16 个标题且与模板一致；status 与 content 一致（confirmed 非 null，unknown 为 null）。
3. `files` 覆盖程序包内（`document/` 除外）的全部文件。
4. `dependencies` 与 `publicApis` 中没有重复的对象。
5. md 中没有 `{{` 和 `<!--`；标题与顺序同模板。
6. md 中 `- [ ] 要確認` 的数量等于 unknown 字段数。
7. Mermaid 通过解析器检查；没有解析工具时人工核对语法。

### 向用户报告（简短，不要把 json、md 的内容贴回对话）

- 生成或更新了哪两个文件。
- unknown 字段清单和各自的 reason。
- `unclassified` 的文件。
- 7.2 中被覆盖的人工改动。
- 7.1 中"AI 现在可以判断"的项。

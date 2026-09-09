---
name: fast-context
description: 用于回答代码相关问题、跨模块定位实现、追踪代码链路，以及判断何时使用 rg/git grep 或 fast-context 语义搜索。包含 fast-context CLI 推荐参数、使用时机和代码搜索协作策略。
---

# Fast Context

用于代码库检索和实现链路定位。回答任何代码相关问题前，必须先调用搜索工具定位代码，禁止凭经验猜测代码位置。

## 使用时机

优先级如下：

1. 已有明确方向时，直接用精确搜索。如果用户给出了文件名、函数名、类名、报错文本、接口路径、配置键、数据库字段、测试名等明确标识符，优先使用 `rg`、`git grep`、`sed`、`nl` 追踪代码。
2. 方向不清或需要跨模块定位时，使用 `fast-context`。适用于“某个功能在哪里实现”“这条链路涉及哪些文件”“删除保护怎么做”“状态机怎么串起来”“数据流从哪里到哪里”等语义问题。
3. 精确标识符、文本、文件名优先 `rg`。例如函数名、类名、常量、错误码、API path、数据库字段、测试名，先用 `rg`；必要时再用 `fast-context` 补全上下游关系。
4. 只搜 Git 跟踪文件时用 `git grep`。适合排除生成物、缓存、依赖目录和未跟踪临时文件。
5. 查历史来源时用 `git log -S/-G`。当问题涉及某段代码何时引入、行为何时变化、谁改过某个判断时，再查历史。

不要为了使用 `fast-context` 而使用 `fast-context`。有明确落点时，精确搜索更快、更可靠。

## 安装与鉴权

首次安装、找不到 `fast-context` 命令、用户要求配置或导入凭据，或搜索出现缺少 Key、401/403 等鉴权错误时，读取 [安装与鉴权参考](references/setup.md)。其中包含 npm 安装命令、`auth import/set/status/clear` 的选择、凭据优先级和排查步骤；正常搜索无需预先执行导入。

## 推荐用法

常规命令：

```bash
fast-context search "自然语言问题" \
  --project /path/to/project \
  --tree-depth 3 \
  --max-turns 3 \
  --max-results 8 \
  --quiet
```

推荐默认参数：

```text
max_results: 8
max_turns: 3
tree_depth: 3
quiet: true
```

CLI 的目录树深度和搜索轮数默认均为 `3`，可省略这两个参数；`FC_MAX_TURNS` 可覆盖默认轮数，显式 `--max-turns` 优先。结果不足时，优先把 `--max-turns` 提高到 `4-5`。目录树深度按下面的预览结果选择。

不熟悉仓库结构时，调用前可用 `rg --files <相关目录> | head -n 80` 快速预览部分文件路径，判断关键模块位于多少层、同层目录是否很多。预览只是结构抽样，无需为了选参数遍历或统计整个仓库；已有目录信息时直接复用。

以 `--project` 为根，选择足以展示相关模块和关键文件的深度（`1-6`）。浅层结构可用 `1-2`；多层嵌套使关键结构在默认 `3` 层内不可见时，可用 `4-6`。同时考虑展开后的目录树规模：分支很多时，优先缩小 `--project` 或添加排除项，必要时再降低深度。不要仅按仓库大小决定深度，也不必展开到仓库最深层。目录树超过 250 KiB 时程序会自动降低深度。

## 参数说明

- `--project <path>`：搜索根目录。尽量缩到相关子目录，减少无关搜索结果和远端 payload。
- `--tree-depth <n>`：发送给模型的初始目录树深度，不限制后续搜索深度。默认 `3`，范围 `1-6`；根据预览中相关模块的层级和目录树展开规模调整，深度相对于 `--project` 计算。
- `--max-turns <n>`：搜索轮数预算，模型可提前回答。默认 `3`；复杂链路或结果不足时用 `4-5`；快速粗查可用 `1-2`。
- `--max-results <n>`：最多返回文件数。默认推荐 `8`；聚焦修改可用 `3-5`；宽泛梳理可用 `10-15`。
- `--quiet`：关闭进度日志，减少对输出的干扰。日常使用建议开启。
- `--exclude <pattern>`：排除目录、文件或 glob，可重复传入。常用：`--exclude node_modules --exclude dist --exclude build --exclude coverage`。
- `--json`：需要结构化结果或后续脚本处理时使用；否则使用默认文本输出。
- `--timeout-ms <n>`：远端请求超时时间。只有在无法继续缩小 `--project` 或降低搜索规模时才提高。

## 工作流

1. 先判断用户问题是否包含明确标识符。
2. 有明确标识符时，先用 `rg` 或 `git grep` 定位。
3. 没有明确落点，或需要理解跨模块链路时，结合已有目录信息或快速预览选择 `--project`、`--tree-depth` 和排除项，再用 `fast-context search`。
4. `fast-context` 返回文件、行号范围、带行号的正文和 grep 关键词。正文共用 50,000 字符预算，按返回文件数平分；超出预算时只保留头尾完整行，中间用 `...` 标记，绝不行内截断。优先使用已返回的正文；遇到截断、读取失败或需要补充上下文时，再用 `sed`、`nl`、`rg` 读取和验证相关代码。JSON 中 `content_truncated` 表示因预算截断，`content_error` 表示正文读取失败；分离的命中范围之间也会用 `...` 分隔。
5. 只有在结论已经被代码定位结果支撑后，才回答或修改代码。

## 示例

语义定位：

```bash
fast-context search "认证逻辑在哪里处理" \
  --project . \
  --tree-depth 3 \
  --max-turns 3 \
  --max-results 8 \
  --quiet
```

缩小到子目录：

```bash
fast-context search "删除保护是怎么实现的" \
  --project ./src \
  --tree-depth 3 \
  --max-turns 3 \
  --max-results 8 \
  --exclude dist \
  --quiet
```

大仓库中已定位到订单子项目，且预览确认其关键文件就在第一层：

```bash
fast-context search "订单状态机在哪里串起来" \
  --project ./services/orders \
  --tree-depth 1 \
  --max-turns 3 \
  --max-results 8 \
  --exclude node_modules \
  --exclude build \
  --quiet
```

## 子代理使用条件

仅当需要读取 10 个以上文件交叉比对，或多轮搜索会明显撑爆上下文时，才启动子代理。普通定位和少量文件追踪不要启动子代理。

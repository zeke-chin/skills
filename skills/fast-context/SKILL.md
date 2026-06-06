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

## 推荐用法

常规命令：

```bash
fast-context search "自然语言问题" \
  --project /path/to/project \
  --tree-depth 2 \
  --max-turns 2 \
  --max-results 8 \
  --quiet
```

推荐默认参数：

```text
max_results: 8
max_turns: 2
tree_depth: 2
quiet: true
```

结果不足时，优先把 `--max-turns` 提高到 `3`。如果结果明显漏掉深层目录，再把 `--tree-depth` 提高到 `3`。

## 参数说明

- `--project <path>`：搜索根目录。尽量缩到相关子目录，减少无关搜索结果和远端 payload。
- `--tree-depth <n>`：发送给模型的目录树深度。默认推荐 `2`；超大仓库用 `1`；小仓库或深层目录较多时用 `3`。
- `--max-turns <n>`：搜索轮数。默认推荐 `2`；复杂链路或结果不足时用 `3`；快速粗查可用 `1`。
- `--max-results <n>`：最多返回文件数。默认推荐 `8`；聚焦修改可用 `3-5`；宽泛梳理可用 `10-15`。
- `--quiet`：关闭进度日志，减少对输出的干扰。日常使用建议开启。
- `--exclude <pattern>`：排除目录、文件或 glob，可重复传入。常用：`--exclude node_modules --exclude dist --exclude build --exclude coverage`。
- `--json`：需要结构化结果或后续脚本处理时使用；否则使用默认文本输出。
- `--timeout-ms <n>`：远端请求超时时间。只有在无法继续缩小 `--project` 或降低搜索规模时才提高。

## 工作流

1. 先判断用户问题是否包含明确标识符。
2. 有明确标识符时，先用 `rg` 或 `git grep` 定位。
3. 没有明确落点，或需要理解跨模块链路时，用 `fast-context search`。
4. `fast-context` 返回文件、行号范围和 grep 关键词后，再用 `sed`、`nl`、`rg` 读取和验证代码。
5. 只有在结论已经被代码定位结果支撑后，才回答或修改代码。

## 示例

语义定位：

```bash
fast-context search "认证逻辑在哪里处理" \
  --project . \
  --tree-depth 2 \
  --max-turns 2 \
  --max-results 8 \
  --quiet
```

缩小到子目录：

```bash
fast-context search "删除保护是怎么实现的" \
  --project ./src \
  --tree-depth 2 \
  --max-turns 3 \
  --max-results 8 \
  --exclude dist \
  --quiet
```

大仓库搜索：

```bash
fast-context search "订单状态机在哪里串起来" \
  --project ./services/orders \
  --tree-depth 1 \
  --max-turns 2 \
  --max-results 8 \
  --exclude node_modules \
  --exclude build \
  --quiet
```

## 子代理使用条件

仅当需要读取 10 个以上文件交叉比对，或多轮搜索会明显撑爆上下文时，才启动子代理。普通定位和少量文件追踪不要启动子代理。

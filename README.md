# opencode-prompt-polisher

OpenCode plugin: automatically polishes your prompts using conversation context.

## Example

**Input** (what you write):

```
帮我写个搜索表格
```

**Output** (filled into the input box after polishing):

```
基于现有项目（React + TypeScript），写一个支持服务端搜索的通用表格组件。需求：
1. 搜索表单支持关键词模糊匹配和状态下拉筛选
2. 表格列支持点击排序，默认按创建时间倒序
3. 分页使用后端返回的 total/page/pageSize
4. 使用 TypeScript interface 定义 props 和数据类型
```

The plugin automatically adds stack, requirement details, typing requirements, and other context-derived constraints. (Examples above use Chinese prompts; the plugin works in any language — see the `default` rule.)

## Features

- **Context-aware** — injects recent conversation messages (file paths, error messages, tech stack) as polishing evidence
- **Safe sub-agent** — runs in a hidden sub-agent with an empty toolset; it never executes your request for you
- **Configurable** — model, context window, intensity, and conditional rules all live in `polish.jsonc`
- **Hot reload** — config edits apply without restarting OpenCode
- **Conditional rules** — keyword hits inject hard constraints (e.g. "SQL" → use CTEs)

## Installation

### npm (recommended)

Edit `~/.config/opencode/opencode.json`:

```jsonc
{
  "plugin": [
    "opencode-prompt-polisher"
  ]
}
```

Restart OpenCode.

### Local development

```jsonc
{
  "plugin": [
    "C:\\path\\to\\opencode-prompt-polisher"
  ]
}
```

Then:

```powershell
cd path/to/opencode-prompt-polisher
npm install
npm run build
```

Requires opencode v2.0 or later.

## Updating

npm installs: re-run the install command or `npm update opencode-prompt-polisher`; local development: `git pull`, then `npm run build` again. Restart OpenCode after updating. There is no built-in auto-update.

## Uninstall

Remove the plugin entry from your OpenCode config (`npm remove opencode-prompt-polisher` for npm installs), restart OpenCode.

## Commands

| Command | Behavior |
|---|---|
| `/polish <prompt>` | Fills the polished result into the input box; you edit or send |
| `/polish-send <prompt>` | Sends the polished result automatically |

### Examples

```
/polish 这个函数性能有问题
```

→ The polished prompt appears in the input box with a "Polish Ready" notice. Send it directly or edit first.

```
/polish-send 写一个二分查找
```

→ Polished and sent directly, no further action needed.

## Configuration

Create `~/.config/opencode/polish.jsonc`:

```jsonc
{
  // Model, format "provider/model-id".
  // Routed internally through OpenCode; no separate apiKey needed.
  "model": "opencode/deepseek-v4-flash-free",

  // Context extraction settings
  "context": {
    "maxMessages": 6,           // inject the most recent N messages
    "maxCharsPerMessage": 500   // truncate each message to this length
  },

  // Intensity (config-file only; never appears in the polish instruction)
  // "light"  - minimal edits, disambiguation only
  // "medium" - standard polish (default)
  // "heavy"  - aggressive rewrite, maximum added detail
  "intensity": "medium",

  // Conditional rules: when the user prompt contains a keyword
  // (case-insensitive substring match), the matching rule is injected into
  // the polish agent's user message as a hard constraint.
  // `default` always applies; multiple `patterns` can hit at once.
  "rules": {
    "default": [
      "保持与用户原 prompt 相同的语言（中文保持中文，英文保持英文，技术术语保持英文）"
    ],
    "patterns": [
      {
        "match": ["code", "function", "实现", "写一个", "implement", "build", "refactor", "test", "tests", "spec", "测试", "单元测试", "覆盖"],
        "rule": "代码/测试类 prompt：优先 TypeScript、提前返回、避免不必要的 try/catch、明确函数签名；测试要注明框架（jest/vitest/pytest 等），覆盖正常/边界/异常路径"
      },
      {
        "match": ["sql", "database", "query", "数据库", "查询"],
        "rule": "SQL 类 prompt：使用标准 SQL、CTE 优先于嵌套子查询、参数化用户输入、注明目标 DBMS"
      },
      {
        "match": ["doc", "documentation", "README", "文档", "教程"],
        "rule": "文档类 prompt：使用 Markdown 标题层级、配代码示例、长文档加目录"
      }
    ]
  }
}
```

Works without a config file too, using built-in defaults.

## How it works

1. The `command.execute.before` hook intercepts `/polish` commands
2. Reads the current session's most recent N messages as context
3. Matches `rules.patterns` by keyword, injecting hits as hard constraints into the user message
4. Calls the LLM in a hidden `polish` sub-agent (empty toolset)
5. Writes the result back to the input box (`/polish`) or sends it (`/polish-send`)

### Rule matching

Keywords in `rules.patterns[].match` are case-insensitive substring matches:

| User prompt contains | Hard constraint applied |
|---|---|
| `code` / `function` / `写一个` / `test` / `测试` | Prefer TypeScript, early returns, name the test framework… |
| `sql` / `数据库` / `query` | CTE-first, parameterized inputs… |
| `doc` / `README` / `文档` | Markdown heading levels, code examples… |
| No match | Only the `default` rule |

The LLM treats matched rules as inviolable hard constraints — the polished result must reflect them.

## Development

```powershell
npm run build              # tsup bundle
npm run typecheck          # tsc typecheck
node tests/match-rules.test.mjs   # rule-matching tests (8 cases)
```

Release flow (pushing a tag auto-publishes to npm):

```powershell
npm version patch   # 0.1.0 → 0.1.1
git push
git push --tags     # triggers GitHub Actions
```

GitHub Actions config lives in `.github/workflows/publish.yml`. The repo needs an `NPM_TOKEN` secret.

## License

MIT

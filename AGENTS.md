# 供在此仓库中工作的编码智能体使用

更改任何内容前请先阅读本文。以下每一行均已于 2026-11-09 使用实时 API、`@brightdata/sdk` 1.2.0 或 Bright Data CLI 0.4.1 验证。如果 API 参考页面与实时 API 不一致，此处以实时 API 为准。

## 身份验证

- `token()` 会读取 `BRIGHTDATA_API_TOKEN`，然后读取存储在 `brightdata-cli/credentials.json` 中的密钥 `bdata login`（SDK 读取的也是同一个文件）。不支持 `.env`；Node 20 可通过 `node --env-file=.env` 加载该文件。设置变量或登录一次，是唯一需要人工完成的步骤；绝不要将密钥粘贴到代码中。
- README 中的原始 HTTP 快速入门仅会读取 `BRIGHTDATA_API_TOKEN`，无法识别 CLI 登录。

## API 的行为方式

- 仅使用一个数据集 `gd_m7aof0k82r803d5bjm`，且只能进行采集。每项输入都需要 `url: "https://chatgpt.com/"` 和一个 `prompt`。
- JavaScript SDK 对其进行了封装，但其输入类型中没有 `require_sources`。此软件包使用内置的 `fetch` 调用 `https://api.brightdata.com/datasets/v3`，且没有依赖项。请保持这种方式。测试通过 `transport` 模拟网络。
- 每次调用都是异步任务：`/trigger`、轮询 `/progress/<id>`，然后执行 `/snapshot/<id>?format=json`。获得答案需要 44 至 282 秒；一次包含 `web_search` 和 `require_sources` 的调用耗时 667 秒。`TIMEOUT_MS` 为 900000。
- 如果状态轮询因网络错误失败，会持续重试直至截止时间。HTTP 错误仍会导致拒绝。
- 始终发送 `include_errors=true`。
- 截至 2026-11-09 的实际行为：
  - 如果没有 `web_search`，`citations` 可能是 `null`，而不是 `[]`（某次运行），也可能是完整列表（另一次运行）。请针对 `null` 做好防护。
  - `web_search_triggered` 不会遵循输入：某个包含十条引用的答案中为 `false`，而在未提供 `web_search` 的提示词中为 `true`。
  - 每条记录中的 `model` 都是 `null`。
  - `geolocation`（"40.7484,-73.9857"，曼哈顿中城）返回了有关休斯敦的答案，并显示“由于无法获取你的精确位置”。不要在文档中将其描述为可用功能。
  - `answer_text_markdown` 采用 html2text 风格：Setext 标题、管道表格、在单独的语言名称行下缩进四个空格的代码，绝不使用围栏代码块。
  - `answer_html` 是完整页面，其中一次结果约为 1.4 MB，且全部位于一行。正因如此，README 摘录会将每行截断至 160 个字符；测试也会执行相同操作。
- 架构可能在不另行通知的情况下发生变化。绝不要硬编码字段列表。
- 完整文档索引，每个条目对应一个 `.md` 页面：https://docs.brightdata.com/llms.txt

## 在此仓库中工作

- `npm test` 可离线运行且不需要令牌。`npm run lint` 必须通过。
- CI 会在空白机器上执行 README 自身的命令，并通过准确的 H2 文本查找各部分。如果重命名标题，也要在 ci.yml 中重命名。
- 每周工作流会针对真实 API 执行 README 中的每个围栏代码块，同一次运行还会执行规模较小的实时检查。代码片段必须完整且可单独粘贴运行，显示的输出也必须是真实结果。
- README 中的字段表位于 `<!-- fields:start -->` 和 `<!-- fields:end -->` 之间，由 `.github/scripts/field_tables.mjs` 每周根据 https://docs.brightdata.com/scrapers-full.json 重新生成。不要手动编辑。
- README 顶部的“最后验证”徽章行会由每周运行重写。不要手动编辑。
- 标题按搜索需求编写，例如“适用于 Node.js 的 ChatGPT 爬虫工具”“如何抓取……”。添加章节时请保持这种模式。
- 保持精简。除状态轮询外，不要添加重试、去重、调度、数据库、异步示例或并发功能。

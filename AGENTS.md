# 供在此仓库中工作的编码智能体使用

更改任何内容前请先阅读本文。以下每一行均已于 2026-11-09 对照实时 API、`@brightdata/sdk` 1.2.0 或 Bright Data CLI 0.4.1 验证。如果 API 参考页面与实时 API 不一致，此处以实时 API 为准。

## 身份验证

- `token()` 先读取 `BRIGHTDATA_API_TOKEN`，然后读取存储在 `brightdata-cli/credentials.json` 中的密钥 `bdata login`（SDK 也读取同一文件）。不支持 `.env`；Node 20 可通过 `node --env-file=.env` 加载该文件。设置变量或运行一次登录，是唯一需要人工完成的步骤；切勿将密钥粘贴到代码中。
- README 中的原始 HTTP 快速入门仅读取 `BRIGHTDATA_API_TOKEN`。它无法获取 CLI 登录信息。

## API 的行为方式

- 仅使用一个数据集 `gd_m7aof0k82r803d5bjm`，且仅支持采集。每个输入都需要 `url: "https://chatgpt.com/"` 和一个 `prompt`。
- JavaScript SDK 对其进行了封装，但其输入类型没有 `require_sources`。此软件包使用内置的 `fetch` 调用 `https://api.brightdata.com/datasets/v3`，且没有依赖项。请保持这种状态。测试通过 `transport` 模拟网络。
- 每次调用都是异步任务：`/trigger`、轮询 `/progress/<id>`，然后 `/snapshot/<id>?format=json`。答案耗时 44 至 282 秒；一次使用 `web_search` 和 `require_sources` 的调用耗时 667 秒。`TIMEOUT_MS` 为 900000。
- 如果状态轮询因网络错误失败，将持续重试直至截止时间。HTTP 错误仍会导致拒绝。
- 始终发送 `include_errors=true`。
- 截至 2026-11-09 的实时行为：
  - 如果没有 `web_search`，`citations` 可能是 `null`，而不是 `[]`（某次运行），也可能是完整列表（另一次运行）。请针对 `null` 做防护。
  - `web_search_triggered` 不会与输入保持一致：一个包含十条引用的答案中为 `false`，而在没有 `web_search` 的情况下发送的提示词中为 `true`。
  - 每条记录的 `model` 都是 `null`。
  - `geolocation`（"40.7484,-73.9857"，曼哈顿中城）得到的是关于休斯敦的答案，并显示“由于无法获得你的精确位置”。不要在文档中将其描述为可用。
  - `answer_text_markdown` 采用 html2text 风格：使用 setext 标题、竖线表格，代码位于单独语言名称行下方并缩进四个空格，从不使用围栏代码块。
  - `answer_html` 是完整页面，单行约 1.4 MB。正因如此，README 摘录将每行截断为 160 个字符；测试也采用同样的处理。
- 架构可能随时更改。切勿硬编码字段列表。
- 完整文档索引中每个条目对应一个 `.md` 页面：https://docs.brightdata.com/llms.txt

## 在此仓库中工作

- `npm test` 可离线运行，不需要令牌。`npm run lint` 必须通过。
- CI 会在空机器上执行 README 中的命令，并按 H2 标题的确切文本查找对应章节。重命名标题时，也要在 ci.yml 中同步修改。
- 每周工作流会针对真实 API 执行 README 中的每个围栏代码块，每日工作流则运行规模较小的实时检查。代码片段必须完整且可直接独立粘贴运行，其显示的输出必须真实。
- README 中的字段表位于 `<!-- fields:start -->` 和 `<!-- fields:end -->` 之间，由 `.github/scripts/field_tables.mjs` 每天根据 https://docs.brightdata.com/scrapers-full.json 重新生成。请勿手动编辑。
- README 顶部的“最后验证”徽章行会由每日运行重写。请勿手动编辑。
- 标题针对搜索进行了优化，例如“用于 Node.js 的 ChatGPT 爬虫工具”“如何抓取……”。添加章节时请保持这种模式。
- 保持精简。除状态轮询外，不要添加重试、去重、调度、数据库、异步示例或并发功能。

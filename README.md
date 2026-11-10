# 用于 Node.js 和 JavaScript 的 ChatGPT 爬虫工具

使用 Node.js 和 JavaScript 将 ChatGPT 答案、引用和来源抓取为 JSON。跟踪品牌在 ChatGPT 中的可见度。无需 OpenAI API 密钥。基于 [Bright Data ChatGPT 爬虫工具 API](https://www.bright.cn/products/web-scraper/chatgpt?utm_source=github&utm_medium=referral&utm_content=chatgpt-scraper-node) 构建。新账户每月可获得 5,000 个免费额度。

[![实时检查](https://github.com/bright-cn/chatgpt-scraper-node/actions/workflows/live.yml/badge.svg)](https://github.com/bright-cn/chatgpt-scraper-node/actions/workflows/live.yml) [![最后验证](https://img.shields.io/badge/last%20verified-9%20Nov%202026-brightgreen)](https://github.com/bright-cn/chatgpt-scraper-node/actions/workflows/live.yml) <!-- verified: rewritten by the daily run -->

[快速入门](#如何使用-nodejs-抓取-chatgpt-结果) · [无需 OpenAI 密钥](#无需-openai-api-密钥即可抓取-chatgpt) · [CLI](#批量-chatgpt-提示词抓取-cli) · [引用](#跟踪某个提示词的-chatgpt-引用和品牌可见度) · [端点](#抓取-chatgpt-答案引用和来源) · [数据](#chatgpt-答案数据字段json) · [错误](#chatgpt-爬虫工具错误) · [MCP 和 Claude Code](#chatgpt-mcp-服务器和-claude-code-技能) · [免费额度](#这个-chatgpt-爬虫工具免费吗) · [文档](https://docs.brightdata.com/products/scrapers/chatgpt/introduction) · [支持](#支持)

通过 JavaScript 获取 JSON 格式的 ChatGPT 答案：发送提示词，即可获得 ChatGPT 在其自身界面中显示的答案，包括文本、Markdown 和 HTML，以及它引用的网站；还可按国家/地区获取结果、启用网页搜索并发送后续提示词。专为 ChatGPT 可见度跟踪和 GEO 打造：了解 ChatGPT 提及和推荐哪些品牌，以及引用哪些来源。无需 OpenAI API 密钥、浏览器或依赖项。基于 [Bright Data ChatGPT 爬虫工具 API](https://www.bright.cn/products/web-scraper/chatgpt?utm_source=github&utm_medium=referral&utm_content=chatgpt-scraper-node) 构建。

这里还提供：用于处理提示词列表的单命令 CLI，以及完全不需要 JavaScript 的 [Bright Data CLI](#chatgpt-mcp-服务器和-claude-code-技能)。

## 如何使用 Node.js 抓取 ChatGPT 结果

需要 Node 20 或更高版本，无需安装任何内容：这里使用内置的 `fetch`。请将其保存为 `.mjs` 文件，因为它使用顶层 `await`。

```bash
export BRIGHTDATA_API_TOKEN=YOUR_API_KEY
```

从 [Bright Data 控制面板](https://www.bright.cn/cp/setting/users)获取令牌。还没有账户？[创建账户](https://www.bright.cn/cp/start)；新账户每月可获得[5,000 个免费额度](https://docs.brightdata.com/general/account/billing-and-pricing/free-tier)。

```javascript
const API = "https://api.brightdata.com/datasets/v3";
const HEADERS = {
  Authorization: `Bearer ${process.env.BRIGHTDATA_API_TOKEN}`,
  "Content-Type": "application/json",
};

async function call(path, body) {
  const init = body === undefined ? { headers: HEADERS } : { method: "POST", headers: HEADERS, body: JSON.stringify(body) };
  const response = await fetch(API + path, init);
  if (!response.ok) throw new Error(`HTTP ${response.status}: ${await response.text()}`);
  return response.json();
}

const prompt = {
  url: "https://chatgpt.com/",
  prompt: "Best CRM software for small businesses",
  country: "US",
  web_search: true,
};
const job = await call("/trigger?dataset_id=gd_m7aof0k82r803d5bjm&include_errors=true", [prompt]);
while (!["ready", "failed"].includes((await call(`/progress/${job.snapshot_id}`)).status)) {
  await new Promise((done) => setTimeout(done, 10_000));
}
const [answer] = await call(`/snapshot/${job.snapshot_id}?format=json`);
if (answer.error) throw new Error(answer.error);
console.log(answer.answer_text.slice(0, 120));
for (const citation of (answer.citations ?? []).slice(0, 3)) console.log("cited:", citation.url);
```

```
The best CRM software for a small business depends on whether you need to manage customers, track sales, automate follow
cited: https://www.hubspot.com/products/crm?utm_source=chatgpt.com
cited: https://www.zoho.com/crm/?utm_source=chatgpt.com
cited: https://www.pipedrive.com/?utm_source=chatgpt.com
```

预计需要一到五分钟：API 会向 ChatGPT 提问，循环则等待答案。一次启用网页搜索和来源的运行耗时十一分钟。按[每个答案](https://www.bright.cn/pricing/web-scraper)计费。

该代码片段中有三项不是可选的。

`url` 必须是 `https://chatgpt.com/`。爬虫工具要求每个输入都包含该字段；提示词放在 `prompt` 中。

`web_search: true` 会要求 ChatGPT 搜索网页，这通常会带来引用。如果没有该字段，`citations` 可能返回 `null`，而不是空列表，因此循环对此进行了防护。

`include_errors=true` 会让 API 将失败的提示词报告为包含 `error` 键的行。API 默认关闭此功能。

Bright Data [JavaScript SDK](https://github.com/bright-cn/bright-data-sdk-js) 封装了这个爬虫，但其输入类型没有 `require_sources`，因此无法要求 ChatGPT 提供引用。这就是此处及本软件包使用纯 HTTP 的原因。

## 无需 OpenAI API 密钥即可抓取 ChatGPT

ChatGPT 爬虫工具 API 不需要 OpenAI 账户、OpenAI API 密钥或 ChatGPT 登录信息。唯一的凭据是 Bright Data API 密钥，通过 `BRIGHTDATA_API_TOKEN` 设置。此软件包及其命令也能找到由 `bdata login` 存储的密钥。

它返回用户在 ChatGPT 本身看到的内容，而不是 OpenAI API 返回的内容：包括答案及其引用、链接、购物和地图区块，并按照 ChatGPT 针对相应国家/地区的渲染方式呈现。

## 批量 ChatGPT 提示词抓取 CLI

此仓库中的命令会同时对多个提示词运行快速入门流程，并写入一个 JSON 文件。

```bash
npm install -g github:bright-cn/chatgpt-scraper-node
chatgpt-scraper "Best CRM software for small businesses" "Best project management tool for startups" --web-search
```

```
Asking ChatGPT 2 prompt(s) from US, with web search
Usually one to five minutes each. Billed per answer, 5,000 free credits a month.
asking  "Best CRM software for small businesses"...
got     "Best CRM software for small businesses": 3,335 characters, 11 citations
asking  "Best project management tool for startups"...
got     "Best project management tool for startups": 4,842 characters, 9 citations

Saved 2 answers as JSON to chatgpt.json
```

在终端中，`asking` 行会被原地更新的加载动画和计时器替代，让你可以看到它正在运行以及已经运行了多长时间。

```
--country CC   two-letter country ChatGPT answers from, default US
--web-search   ask ChatGPT to search the web before answering
--out PATH     output file, default chatgpt.json
```

该命令会在 `BRIGHTDATA_API_TOKEN` 中查找令牌；如果你运行过一次 `npx -p @brightdata/cli bdata login`，它也会从已存储的登录信息中查找，以供你以及该终端中的任何编码智能体使用。它不会读取 `.env` 文件；Node 会通过 `node --env-file=.env` 为其加载该文件。

你也可以导入它而不是直接运行，以便为每个提示词获取 `ok` 和 `error`，而不是原始行。此软件包使用 ESM，因此请使用 `import`，不要使用 `require`。`scrape` 不会因单个错误提示词而拒绝；读取 `answer` 前请检查 `ok`：

```javascript
import { scrape } from "@brightdata/chatgpt-scraper-node";

for (const outcome of await scrape(["Best running shoes for beginners", "Best free photo editing app"])) {
  if (outcome.ok) console.log(`${outcome.prompt}: ${outcome.answer.answer_text.slice(0, 60)}`);
  else console.log(`${outcome.prompt} failed: ${outcome.error}`);
}
```

```
Best running shoes for beginners: If you're just getting into running, I recommend shoes that 
Best free photo editing app: Best overall Snapseed – Best free photo editing app Snapseed
```

## 抓取 ChatGPT 答案、引用和来源

只有一个 ChatGPT 爬虫工具，其文档位于 [ChatGPT 爬虫工具 API 文档](https://docs.brightdata.com/products/scrapers/chatgpt/introduction)。下面每个代码片段都是完整的：完成上面的 `npm install` 后可直接原样粘贴。每个片段都会在每周一通过 Actions 运行，规模较小的检查则每隔一天运行一次，顶部徽章显示最新结果。

| 你想要的内容 | 方法 |
| --- | --- |
| ChatGPT 对提示词的回答 | `ask(["prompt one", "prompt two"], { country: "US" })` |
| ChatGPT 引用的网站 | `ask(prompt, { web_search: true, require_sources: true })` |
| 同一聊天中的后续提问 | `ask(prompt, { additional_prompt: "..." })` |
| 来自其他国家/地区的答案 | `ask(prompt, { country: "DE" })` |
| Markdown 格式的答案 | 任何答案上的 `answer_text_markdown` |

`ask` 接受一个提示词或提示词数组。一个数组对应一个任务，而不是每个提示词一个任务。每个答案是一条记录，API 无法回答的提示词会作为包含 `error` 键的行返回。API 接受的任何其他输入均按 API 使用的名称传入，例如 `index`。

### 跟踪某个提示词的 ChatGPT 引用和品牌可见度

```javascript
import { ask } from "@brightdata/chatgpt-scraper-node";

const [answer] = await ask("Best project management software for remote teams", { web_search: true, require_sources: true });
const domains = (answer.citations ?? []).map((c) => new URL(c.url).host);
const unique = [...new Set(domains)].sort();
console.log(domains.length, "citations from", unique.length, "sites:");
for (const domain of unique.slice(0, 5)) console.log(" ", domain, domains.filter((d) => d === domain).length);
```

```
9 citations from 8 sites:
  asana.com 2
  basecamp.com 1
  clickup.com 1
  monday.com 1
  thedigitalprojectmanager.com 1
```

`require_sources` 要求 ChatGPT 提供引用，`web_search` 则让它进行搜索。`citations` 保存每个被引用的 `url` 及其 `title`；`links_attached` 保存答案文本中的链接。每天询问相同的提示词，是 GEO 工具跟踪引用份额随时间变化的方式。

### 向 ChatGPT 发送后续提示词

```javascript
import { ask } from "@brightdata/chatgpt-scraper-node";

const [answer] = await ask("Name three popular sourdough bread recipes", {
  additional_prompt: "Which of them is easiest for a beginner? One sentence.",
});
console.log("first: ", answer.answer_text.slice(0, 80));
console.log("second:", answer.additional_answer_text.slice(0, 80));
```

```
first:  Here are three popular sourdough bread recipes worth trying: 1. Classic Artisan 
second: The classic artisan country sourdough is easiest for a beginner because it uses 
```

后续提示词在同一聊天中运行，因此可以引用第一个答案。其答案会存入 `additional_answer_text`，并包含自己的 `citations_additional_prompt`。

### 按国家/地区比较 ChatGPT 答案

```javascript
import { ask } from "@brightdata/chatgpt-scraper-node";

for (const country of ["US", "DE"]) {
  const [answer] = await ask("Which supermarket chain has the cheapest groceries?", { country, web_search: true });
  console.log(country, "|", answer.answer_text.slice(0, 90));
}
```

```
US | Aldi is generally the cheapest traditional supermarket chain in the U.S. , with Lidl a ver
DE | If you're shopping in Germany , my recommendation is ALDI or Lidl for consistently afforda
```

`country` 是 ChatGPT 作答时所在国家/地区的双字母代码。美国的答案首先提到 Aldi；德国的答案首先提到 ALDI 和 Lidl。API 也接受 `geolocation`，但在 2026 年 11 月 9 日，曼哈顿中城的坐标得到的是关于休斯敦的答案，开头为“由于无法获得你的精确位置”，因此本软件包未将其记录为可用功能。

### 获取 Markdown 格式的 ChatGPT 答案

```javascript
import { writeFile } from "node:fs/promises";

import { ask } from "@brightdata/chatgpt-scraper-node";

const [answer] = await ask("Explain what a REST API is, with one short code example");
const markdown = answer.answer_text_markdown;
await writeFile("answer.md", markdown, "utf8");
const lines = markdown.split("\n");
const headings = lines.filter((line, i) => /^-+$/.test(lines[i + 1] ?? "") && line.trim());
console.log(markdown.length, "characters saved to answer.md; headings:", headings);
```

```
1900 characters saved to answer.md; headings: [
  'How does a REST API work?',
  'Short code example',
  'Key concept to remember'
]
```

`answer_text_markdown` 从渲染后的页面转换而来：标题使用短横线下划线，而不是写成 `#`；表格使用竖线表格；代码位于标明语言的行下方并缩进四个空格，而不是使用围栏代码块。`answer_text` 是同一答案的纯文本版本，`answer_html` 则是完整页面，约 1.4 MB。

### 立即触发，稍后获取

对于较长的提示词列表，不要阻塞进程一小时。触发任务、保存快照 ID，并在准备就绪时获取。快照可在 30 天内下载。

```javascript
import { CHATGPT, fetchSnapshot, status, trigger } from "@brightdata/chatgpt-scraper-node";

const item = { url: "https://chatgpt.com/", prompt: "What is the capital of Australia?", country: "US" };
const snapshot = await trigger(CHATGPT, [item]);
console.log("snapshot:", snapshot);
let state;
while (!["ready", "failed"].includes((state = await status(snapshot)))) {
  await new Promise((done) => setTimeout(done, 10_000));
}
console.log("status:", state);
console.log("fetched:", (await fetchSnapshot(snapshot))[0].answer_text.slice(0, 60));
```

```
snapshot: sd_mw9canbkoi5uoia7a
status: ready
fetched: The capital of Australia is Canberra . 🇦🇺 It is located in
```

截至 2026-11-09，实时 API 为数据集 `gd_m7aof0k82r803d5bjm` 接受的输入字段：

| 输入 | 必填 | 作用 |
| --- | --- | --- |
| `url` | 是 | 始终为 `https://chatgpt.com/` |
| `prompt` | 是 | 问题 |
| `country` | 否 | ChatGPT 作答时所在国家/地区的双字母代码 |
| `web_search` | 否 | `true` 要求 ChatGPT 搜索网页 |
| `require_sources` | 否 | `true` 要求 ChatGPT 引用来源 |
| `additional_prompt` | 否 | 同一聊天中的后续提问 |
| `index` | 否 | 你自己的编号，会在记录中返回 |
| `geolocation` | 否 | `"lat,long"`；在我们的测试中未生效 |

`bdata pipelines info gd_m7aof0k82r803d5bjm collect_by_url` 会输出 API 验证的输入字段。

## ChatGPT 答案数据字段（JSON）

大多数人希望从答案中获取的字段：

```
prompt  answer_text  answer_text_markdown  citations  links_attached  country  web_search_triggered
```

`web_search_triggered` 不会与 `web_search` 输入保持一致：2026 年 11 月 9 日，一个包含十条引用的答案中该字段为 `false`，而没有 `web_search` 的提示词中为 `true`。每条记录的 `model` 都是 `null`。代码没有硬编码任何字段列表。API 返回的所有内容都会进入每条记录以及命令生成的文件中。

<!-- fields:start -->
每天根据已发布的架构 [scrapers-full.json](https://docs.brightdata.com/scrapers-full.json) 重新生成，因此不会过时。记录包含适用于它的字段，以及 `timestamp` 和 `input`；示例文件中的第一个答案有 33 个键。

<details>
<summary>全部 32 个 ChatGPT 答案字段，<code>gd_m7aof0k82r803d5bjm</code></summary>

| 字段 | 类型 | 描述 |
| --- | --- | --- |
| `url` | URL | ChatGPT 页面 URL |
| `prompt` | 文本 | 输入的提示词 |
| `answer_html` | 文本 | ChatGPT 搜索的答案 |
| `answer_text` | 文本 | 纯文本格式的答案文本 |
| `links_attached` | 数组 | 响应文本中包含的超链接，以及它们的位置和链接文字。 |
| `citations` | 数组 | 答案中直接引用的来源，通常链接到具体事实或声明。 |
| `country` | 国家/地区 | 执行搜索时所在的国家/地区 |
| `is_map` | 布尔值 | 是否生成了地图包结果 |
| `references` | 数组 | 在“参考资料”部分下分组的链接 |
| `shopping` | 数组 | ChatGPT“购物”面板中的商品推荐卡片 |
| `shopping_visible` | 布尔值 | 此答案是否显示了“购物”面板 |
| `index` | 数字 | 索引列允许你为每个爬取请求分配唯一 ID，以便跟踪。 |
| `answer_text_markdown` | html2markdown | Markdown 格式的答案文本。 |
| `web_search_triggered` | 布尔值 | ChatGPT 返回答案时是否触发了网页搜索 |
| `additional_prompt` | 文本 | additional_prompt 是收到第一个答案后发送的后续输入，用于澄清、扩展或完善初始响应。系统处理此提示词，并将结果作为 additional_answer_text 返回。 |
| `additional_answer_text` | 文本 | additional_answer_text 是系统对 additional_prompt 的响应，发生在初始交互之后。它会根据用户的后续输入提供额外信息、澄清内容或改进后的答案。 |
| `map` | 数组 | 地图包结果 |
| `search_sources` | 数组 | ChatGPT 用于收集信息的外部网页列表，通常显示在响应底部。 |
| `search_sources_more` | 数组 | ChatGPT 用于收集信息的外部网页列表，通常显示在响应底部（来自“更多”分区）。 |
| `response_raw` | 文本 | ChatGPT 响应的原始数据 |
| `response_raw_additional_prompt` | 文本 | ChatGPT 对后续提示词响应的原始数据。 |
| `answer_section_html` | 文本 | 答案各部分的 HTML |
| `model` | 文本 | 用于生成答案的模型 |
| `web_search_query` | 数组 | 模型生成的网页搜索查询（确切字符串，按执行顺序排列） |
| `prompt_sent_at` | 日期 | 提示词的确切时间戳 |
| `answer_text_raw` | 文本 | 提取出的原始 answer_text，未进行截断或空白字符规范化（可能包含前导/尾随空格和换行符）。 |
| `business_locations` | 数组 | 请考虑添加新字段，并附上我们可为答案中提及的每个商家位置获取的 JSON 示例。答案包含本地商家结果时返回的商家位置实体数组。每个条目包含商家 ID、坐标、名称、类别、地址、联系信息、评分、营业时间、图片和营业/歇业状态。 |
| `ads` | 数组 | 广告卡片数据数组。 |
| `additional_answer_text_markdown` | html2markdown | Markdown 格式的 additional_answer_text_markdown。 |
| `questionnaire` | 数组 | 问卷 |
| `shopping_additional_prompt` | 数组 | 后续提示词结果中 ChatGPT“购物”面板的商品推荐卡片 |
| `citations_additional_prompt` | 数组 | 后续提示词答案中直接引用的来源，通常链接到具体事实或声明。 |

</details>
<!-- fields:end -->

<details>
<summary>来自 <code>chatgpt-scraper "Best CRM software for small businesses"</code> 的真实输出文件开头</summary>

此处每行均截断为 160 个字符；仅 `answer_html` 一行就有 1.4 MB。

```json
{
  "generated_at": "2026-11-09T14:12:26.652Z",
  "country": "US",
  "prompts": [
    {
      "prompt": "Best CRM software for small businesses",
      "answer": {
        "url": "https://chatgpt.com/?q=Best%20CRM%20software%20for%20small%20businesses",
        "prompt": "Best CRM software for small businesses",
        "answer_html": "<html class=\"x108lcm5 xzx5cyk xojncy5 xy3g5xu x1nktip0 xwbwmuu x1it3fuk xh8yej3 xuu8av0 xeuugli xgs08xk x1javzw2 xfqjyog x1rob05q xvmah...
        "answer_text": "The best CRM software for a small business depends on whether you primarily need to manage customers, track sales leads, automate follow...
        "links_attached": [
          {
            "url": "https://www.hubspot.com/products/crm?utm_source=chatgpt.com",
            "text": "HubSpot website",
            "position": 1
          },
          {
            "url": "https://www.zoho.com/crm/?utm_source=chatgpt.com",
            "text": "Zoho website",
  ...
```

包含一个答案及全部字段的完整文件位于 [examples/sample_output.json](examples/sample_output.json)。

</details>

## ChatGPT 爬虫工具错误

| 你看到的内容 | 含义 |
| --- | --- |
| `API token required but not found.` | 在发送任何请求前以代码 2 退出。请设置令牌，或运行 `bdata login`。 |
| `failed  "prompt": ...` | 以代码 1 退出。API 为该提示词返回了错误行，而不是答案。消息来自 API。 |
| `failed  "prompt": timeout: snapshot ... not ready after 900s` | 以代码 1 退出。请重新运行，或稍后使用 `fetchSnapshot()` 获取快照。 |
| `failed  "prompt": HTTP 401: ...` | 令牌已设置但不正确。 |

任何失败都会以代码 1 退出，因此可以安全地用运行结果控制脚本是否继续。

通过函数调用时，相同情况如下所示：

| 你看到的内容 | 含义 |
| --- | --- |
| `TokenMissing` | 没有 `BRIGHTDATA_API_TOKEN`，也没有 `bdata login`。 |
| `ApiError: HTTP 400: ...validation_error...` | API 不接受的输入。消息会指出具体字段。 |
| 包含 `error` 键的行 | API 针对一个提示词的回答。其他行不受影响。 |
| `ApiError: timeout: ...` | 任务超过了 900 秒的等待时间。将 `{ timeoutMs }` 传给 `run`，或使用 `trigger` 并稍后获取。 |

## ChatGPT MCP 服务器和 Claude Code 技能

无需 JavaScript，也无需安装任何内容。粘贴这两行；第一行会打开一次浏览器，也可以在 SSH 和 CI 中使用 `bdata login --device`：

```bash
npx -p @brightdata/cli bdata login
npx -p @brightdata/cli bdata pipelines run gd_m7aof0k82r803d5bjm collect_by_url --url "https://chatgpt.com/" --prompt "Best CRM software for small businesses" --country US --pretty
```

1 条记录、19 个字段，2026 年 11 月 9 日耗时 75 秒。CLI 会省略值为 null 的字段。记录约为 3 MB 的 JSON，其中大部分是 `answer_html`。

`npx skills add brightdata/skills` 会让 Claude Code、Cursor 和 Codex 掌握这些命令和文档，之后即可使用自然语言操作。完整指南：[面向编码智能体的 Bright Data](https://docs.brightdata.com/quickstart-coding-agent)。

要供托管智能体使用且没有终端？[Bright Data MCP 服务器](https://github.com/bright-cn/brightdata-mcp#which-tool-to-use) 可通过其 `scrapers` 组中的 `search_scrapers` 和 `run_scraper` 运行任何预构建爬虫工具，包括 ChatGPT：

    https://mcp.brightdata.com/mcp?token=YOUR_API_TOKEN&groups=scrapers

智能体也可以自行开设账户，无需注册表单：[智能体注册](https://brightdata.com/auth.md)。Bright Data 连接的其他所有工具，从 LangChain 到 Zapier 和 n8n：[集成](https://docs.brightdata.com/integrations/introduction)。

## 这个 ChatGPT 爬虫工具免费吗？

新的 Bright Data 账户每月可获得[5,000 个免费额度](https://docs.brightdata.com/general/account/billing-and-pricing/free-tier)。ChatGPT 爬虫工具按记录计费，每个答案一条记录。[定价](https://www.bright.cn/pricing/web-scraper)中列出了费率。

## 支持

此仓库中的错误：[提交 issue](https://github.com/bright-cn/chatgpt-scraper-node/issues)，[CONTRIBUTING.md](CONTRIBUTING.md) 说明了需要提供哪些信息。任何有关 API、你的账户或额度的问题，请联系 [Bright Data 支持团队](https://brightdata.zendesk.com/hc/en-us/requests/new)。

## 许可证

[Apache 2.0](LICENSE)。

## 用于 Python 的 ChatGPT 爬虫工具

同一 ChatGPT 爬虫工具也提供 Python 版本：[bright-cn/chatgpt-scraper](https://github.com/bright-cn/chatgpt-scraper)。

## 更多来自 Bright Data 的开源爬虫工具

其他采用相同方式构建并包含实时验证示例的 Node.js 爬虫工具：

- [用于 Node.js 的 URL 转 Markdown（Fetch API）](https://github.com/bright-cn/fetch-api-scraper-node)
- [用于 Node.js 的 YouTube 爬虫工具](https://github.com/bright-cn/youtube-scraper-node)
- [用于 Node.js 的 LinkedIn 爬虫工具](https://github.com/bright-cn/linkedin-scraper-node)

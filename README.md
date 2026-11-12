# 适用于 Node.js 和 JavaScript 的 ChatGPT 抓取工具

使用 Node.js 和 JavaScript 将 ChatGPT 的答案、引用和来源抓取为 JSON。跟踪品牌在 ChatGPT 中的可见性。无需 OpenAI API 密钥。基于 [Bright Data ChatGPT 爬虫工具 API](https://www.bright.cn/products/web-scraper/chatgpt?utm_source=github&utm_medium=referral&utm_content=chatgpt-scraper-node) 构建。新账户每月可获得 5,000 个免费额度。

[![实时检查](https://github.com/bright-cn/chatgpt-scraper-node/actions/workflows/live.yml/badge.svg)](https://github.com/bright-cn/chatgpt-scraper-node/actions/workflows/live.yml) [![最后验证](https://img.shields.io/badge/last%20verified-11%20Nov%202026-brightgreen)](https://github.com/bright-cn/chatgpt-scraper-node/actions/workflows/live.yml) <!-- verified: rewritten by the weekly run -->

[快速入门](#如何使用-nodejs-抓取-chatgpt-结果) · [无需 OpenAI 密钥](#无需-openai-api-密钥即可抓取-chatgpt) · [CLI](#批量-chatgpt-提示词抓取-cli) · [引用](#跟踪某个提示词的-chatgpt-引用和品牌可见性) · [端点](#抓取-chatgpt-答案引用和来源) · [数据](#chatgpt-答案数据字段json) · [错误](#chatgpt-爬虫工具错误) · [MCP 和 Claude Code](#chatgpt-mcp-服务器和-claude-code-技能) · [免费额度](#这个-chatgpt-爬虫工具免费吗) · [文档](https://docs.brightdata.com/products/scrapers/chatgpt/introduction) · [支持](#支持)

以 JavaScript 中的 JSON 形式获取 ChatGPT 答案：发送提示词，即可获取 ChatGPT 在其自身界面中显示的答案，包括文本、Markdown 和 HTML，以及它引用的网站；支持按国家/地区、网络搜索和后续提示词。专为 ChatGPT 可见性跟踪和 GEO 构建：了解 ChatGPT 提及和推荐哪些品牌，以及引用哪些来源。无需 OpenAI API 密钥、浏览器或依赖项。基于 [Bright Data ChatGPT 爬虫工具 API](https://www.bright.cn/products/web-scraper/chatgpt?utm_source=github&utm_medium=referral&utm_content=chatgpt-scraper-node) 构建。

这里还提供：用于处理提示词列表的单命令 CLI，以及完全不需要 JavaScript 的 [Bright Data CLI](#chatgpt-mcp-服务器和-claude-code-技能)。

## 如何使用 Node.js 抓取 ChatGPT 结果

需要 Node 20 或更高版本，无需安装任何内容：这里使用内置的 `fetch`。由于它使用顶层 `await`，请将其保存为 `.mjs` 文件。

```bash
export BRIGHTDATA_API_TOKEN=YOUR_API_KEY
```

从 [Bright Data 控制面板](https://www.bright.cn/cp/setting/users)获取令牌。还没有账户？[创建账户](https://www.bright.cn/cp/start)；新账户[每月可获得 5,000 个免费额度](https://docs.brightdata.com/general/account/billing-and-pricing/free-tier)。

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

预计需要一到五分钟：API 会向 ChatGPT 提问，循环则等待答案。一次启用网络搜索和来源的运行耗时十一分钟。按[每个答案](https://www.bright.cn/pricing/web-scraper)计费。

该代码片段中的三项设置不是可选的。

`url` 必须是 `https://chatgpt.com/`。爬虫工具要求每项输入都包含该字段；提示词放在 `prompt` 中。

`web_search: true` 会要求 ChatGPT 搜索网络，这通常会带来引用。如果没有它，`citations` 可能返回 `null`，而不是空列表，因此循环会对此进行防护。

`include_errors=true` 使 API 将失败的提示词报告为包含 `error` 键的行。API 默认关闭此功能。

Bright Data [JavaScript SDK](https://github.com/bright-cn/sdk-js) 封装了该爬虫，但其输入类型中没有 `require_sources`，因此无法要求 ChatGPT 提供引用。正因如此，此处及本软件包中使用原生 HTTP。

## 无需 OpenAI API 密钥即可抓取 ChatGPT

ChatGPT 爬虫工具 API 不需要 OpenAI 账户、OpenAI API 密钥或 ChatGPT 登录信息。唯一凭据是设置为 `BRIGHTDATA_API_TOKEN` 的 Bright Data API 密钥。此软件包及其命令也能找到由 `bdata login` 存储的密钥。

它返回的是用户在 ChatGPT 自身界面中看到的内容，而不是 OpenAI API 返回的内容：包括答案及其引用、链接、购物和地图区块，并按照 ChatGPT 针对相应国家/地区的呈现方式返回。

## 批量 ChatGPT 提示词抓取 CLI

此仓库中的命令会同时针对多个提示词运行快速入门流程，并写入一个 JSON 文件。

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

在终端中，`asking` 行会被旋转加载图标和持续计时器替代，并原地更新，让你可以看到任务正在运行以及已经运行了多长时间。

```
--country CC   two-letter country ChatGPT answers from, default US
--web-search   ask ChatGPT to search the web before answering
--out PATH     output file, default chatgpt.json
```

该命令会在 `BRIGHTDATA_API_TOKEN` 中查找令牌；如果你曾运行过一次 `npx -p @brightdata/cli bdata login`，则会从已保存的登录信息中查找，供你以及该终端中的任何编码智能体使用。它不会读取 `.env` 文件；Node 会通过 `node --env-file=.env` 为其加载该文件。

你也可以导入它而不是直接运行，以便为每个提示词获取 `ok` 和 `error`，而不是原始行。此软件包使用 ESM，因此请使用 `import`，而不是 `require`。`scrape` 不会因为某个错误提示词而拒绝；读取 `answer` 前请检查 `ok`：

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

这里只使用一个 ChatGPT 爬虫工具，相关说明请参阅 [ChatGPT 爬虫工具 API 文档](https://docs.brightdata.com/products/scrapers/chatgpt/introduction)。以下每个代码片段都是完整的：在上方的 `npm install` 之后直接粘贴即可。它们每周一都会在 Actions 中运行，每隔一天还会执行一次规模较小的检查，顶部徽章显示最新结果。

| 你想要的内容 | 实现方式 |
| --- | --- |
| ChatGPT 对提示词的回答 | `ask(["prompt one", "prompt two"], { country: "US" })` |
| ChatGPT 引用的网站 | `ask(prompt, { web_search: true, require_sources: true })` |
| 同一对话中的后续提问 | `ask(prompt, { additional_prompt: "..." })` |
| 来自其他国家/地区的答案 | `ask(prompt, { country: "DE" })` |
| Markdown 格式的答案 | 任何答案上的 `answer_text_markdown` |

`ask` 接受一个提示词或提示词数组。一个数组对应一个任务，而不是每个提示词各一个任务。每个答案对应一条记录，API 无法回答的提示词会以包含 `error` 键的行返回。API 接受的任何其他输入都按 API 中的名称传入，例如 `index`。

### 跟踪某个提示词的 ChatGPT 引用和品牌可见性

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

`require_sources` 要求 ChatGPT 提供引用，`web_search` 则让它进行查找。`citations` 保存每个被引用的 `url` 及其 `title`；`links_attached` 保存答案文本中的链接。每天询问相同提示词，是 GEO 工具持续跟踪引用份额的方式。

### 向 ChatGPT 提出后续问题

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

后续问题会在同一对话中运行，因此可以引用第一个答案。其答案会写入 `additional_answer_text`，并带有自己的 `citations_additional_prompt`。

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

`country` 是 ChatGPT 作答所在国家/地区的两字母代码。美国版答案首先提到 Aldi；德国版则首先提到 ALDI 和 Lidl。也接受 `geolocation`，但在 2026 年 11 月 9 日，曼哈顿中城的坐标返回了有关休斯敦的答案，并以“由于无法获取你的精确位置”开头，因此本软件包不将其记录为受支持功能。

### 以 Markdown 获取 ChatGPT 的答案

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

`answer_text_markdown` 从渲染后的页面转换而来：标题使用短横线下划线，而不是以 `#` 开头；表格使用管道表格；代码则在语言名称行下缩进四个空格，而不使用围栏。`answer_text` 是同一答案的纯文本版本，`answer_html` 则是完整页面，约 1.4 MB。

### 立即触发，稍后获取

对于很长的提示词列表，不要让进程阻塞一小时。触发任务、保存快照 ID，准备就绪后再获取。快照可下载 30 天。

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

截至 2026-11-09，实时 API 对数据集 `gd_m7aof0k82r803d5bjm` 接受的输入字段：

| 输入 | 必填 | 作用 |
| --- | --- | --- |
| `url` | 是 | 始终为 `https://chatgpt.com/` |
| `prompt` | 是 | 问题 |
| `country` | 否 | ChatGPT 作答所在国家/地区的两字母代码 |
| `web_search` | 否 | `true` 要求 ChatGPT 搜索网络 |
| `require_sources` | 否 | `true` 要求 ChatGPT 引用来源 |
| `additional_prompt` | 否 | 同一对话中的后续提问 |
| `index` | 否 | 你自己的编号，会随记录返回 |
| `geolocation` | 否 | `"lat,long"`；在我们的测试中未生效 |

`bdata pipelines info gd_m7aof0k82r803d5bjm collect_by_url` 会输出 API 验证的输入字段。

## ChatGPT 答案数据字段（JSON）

大多数人希望从答案中获取的字段：

```
prompt  answer_text  answer_text_markdown  citations  links_attached  country  web_search_triggered
```

`web_search_triggered` 不遵循 `web_search` 输入：2026 年 11 月 9 日，某个包含十条引用的答案中其值为 `false`，而在未提供 `web_search` 的提示词中为 `true`。每条记录中的 `model` 都是 `null`。代码没有硬编码任何字段列表。API 返回的所有内容都会写入每条记录以及命令生成的文件中。

<!-- fields:start -->
每周根据已发布的架构 [scrapers-full.json](https://docs.brightdata.com/scrapers-full.json) 重新生成，因此不会过时。记录包含适用于该记录的字段，以及 `timestamp` 和 `input`；示例文件中的第一个答案有 33 个键。

<details>
<summary>全部 32 个 ChatGPT 答案字段，<code>gd_m7aof0k82r803d5bjm</code></summary>

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `url` | url | ChatGPT 页面 URL |
| `prompt` | text | 输入的提示词 |
| `answer_html` | text | ChatGPT 搜索的答案 |
| `answer_text` | text | 纯文本格式的答案 |
| `links_attached` | array | 响应文本中包含的超链接，以及它们的位置和链接文字。 |
| `citations` | array | 答案中直接引用的来源，通常链接到具体事实或主张。 |
| `country` | country | 执行搜索的国家/地区 |
| `is_map` | boolean | 是否生成了地图组件结果 |
| `references` | array | “参考资料”部分下分组显示的链接 |
| `shopping` | array | ChatGPT“购物”面板中的商品推荐卡片 |
| `shopping_visible` | boolean | 此答案是否显示了“购物”面板 |
| `index` | number | 索引列允许你为每个爬取请求分配唯一 ID，以便轻松跟踪。 |
| `answer_text_markdown` | html2markdown | Markdown 格式的答案文本。 |
| `web_search_triggered` | boolean | ChatGPT 返回答案时是否触发了网络搜索 |
| `additional_prompt` | text | additional_prompt 是在收到第一个答案后发送的后续输入，用于澄清、扩展或完善初始回答。系统处理该提示词，并将结果作为 additional_answer_text 返回。 |
| `additional_answer_text` | text | additional_answer_text 是系统对 additional_prompt 的响应，发生在初始交互之后。它会根据用户的后续输入提供附加信息、说明或完善后的答案。 |
| `map` | array | 地图组件结果 |
| `search_sources` | array | ChatGPT 用于收集信息的外部网页列表，通常显示在响应底部。 |
| `search_sources_more` | array | ChatGPT 用于收集信息的外部网页列表，通常显示在响应底部（来自“更多”分区）。 |
| `response_raw` | text | ChatGPT 响应的原始数据 |
| `response_raw_additional_prompt` | text | ChatGPT 对后续提示词响应的原始数据。 |
| `answer_section_html` | text | 答案部分的 HTML |
| `model` | text | 用于生成答案的模型 |
| `web_search_query` | array | 模型生成的网络搜索查询（准确字符串，按执行顺序排列） |
| `prompt_sent_at` | date | 提示词的准确时间戳 |
| `answer_text_raw` | text | 提取出的原始 answer_text，未进行裁剪或空白规范化处理（可能包含前导/尾随空格和换行符）。 |
| `business_locations` | array | 请考虑添加新字段，并附上答案中提及的每个商业地点可获得的 JSON 示例。答案包含本地商家结果时返回的商业地点实体数组。每个条目都包含商家 ID、坐标、名称、类别、地址、联系信息、评分、营业时间、图片及营业/停业状态。 |
| `ads` | array | 广告卡片数据数组。 |
| `additional_answer_text_markdown` | html2markdown | Markdown 格式的 additional_answer_text_markdown。 |
| `questionnaire` | array | 问卷 |
| `shopping_additional_prompt` | array | 后续提示词结果中 ChatGPT“购物”面板里的商品推荐卡片 |
| `citations_additional_prompt` | array | 后续提示词答案中直接引用的来源，通常链接到具体事实或主张。 |

</details>
<!-- fields:end -->

<details>
<summary>真实输出文件的开头，来自 <code>chatgpt-scraper "Best CRM software for small businesses"</code></summary>

此处每行均截断至 160 个字符；仅 `answer_html` 一行就有 1.4 MB。

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

完整文件包含一个带有所有字段的答案：[examples/sample_output.json](examples/sample_output.json)。

</details>

## ChatGPT 爬虫工具错误

| 你看到的内容 | 含义 |
| --- | --- |
| `API token required but not found.` | 在发出任何请求前以状态码 2 退出。请设置令牌，或运行 `bdata login`。 |
| `failed  "prompt": ...` | 以状态码 1 退出。API 为该提示词返回了错误行，而不是答案。消息来自 API。 |
| `failed  "prompt": timeout: snapshot ... not ready after 900s` | 以状态码 1 退出。重新运行，或稍后使用 `fetchSnapshot()` 获取快照。 |
| `failed  "prompt": HTTP 401: ...` | 令牌已设置，但不正确。 |

任何失败都会以状态码 1 退出，因此可安全地用运行结果控制脚本是否继续执行。

在函数中，相同情况如下所示：

| 你看到的内容 | 含义 |
| --- | --- |
| `TokenMissing` | 没有 `BRIGHTDATA_API_TOKEN`，也没有 `bdata login`。 |
| `ApiError: HTTP 400: ...validation_error...` | API 不接受的输入。消息中会指出相应字段。 |
| 包含 `error` 键的行 | API 对某个提示词的回答。其他行不受影响。 |
| `ApiError: timeout: ...` | 任务超过了 900 秒的等待时间。将 `{ timeoutMs }` 传给 `run`，或使用 `trigger` 并稍后获取。 |

## ChatGPT MCP 服务器和 Claude Code 技能

无需 JavaScript，也无需安装任何内容。粘贴这两行；第一行会打开一次浏览器，也可以在 SSH 和 CI 中使用 `bdata login --device`：

```bash
npx -p @brightdata/cli bdata login
npx -p @brightdata/cli bdata pipelines run gd_m7aof0k82r803d5bjm collect_by_url --url "https://chatgpt.com/" --prompt "Best CRM software for small businesses" --country US --pretty
```

2026 年 11 月 9 日的运行结果为 1 条记录、19 个字段，耗时 75 秒。CLI 会省略值为 null 的字段。该记录约为 3 MB 的 JSON，其中大部分是 `answer_html`。

`npx skills add brightdata/skills` 会向 Claude Code、Cursor 和 Codex 说明这些命令与文档，之后即可使用自然语言操作。完整指南：[为你的编码智能体使用 Bright Data](https://docs.brightdata.com/quickstart-coding-agent)。

使用托管式助手且没有终端？[Bright Data MCP 服务器](https://github.com/bright-cn/brightdata-mcp#which-tool-to-use) 可通过其 `scrapers` 组中的 `search_scrapers` 和 `run_scraper` 运行任何预构建爬虫工具，包括 ChatGPT：

    https://mcp.brightdata.com/mcp?token=YOUR_API_TOKEN&groups=scrapers

智能体也可以自行创建账户，无需注册表单：[智能体注册](https://brightdata.com/auth.md)。Bright Data 支持的所有其他连接，从 LangChain 到 Zapier 和 n8n：[集成](https://docs.brightdata.com/integrations/introduction)。

## 这个 ChatGPT 爬虫工具免费吗？

新的 Bright Data 账户[每月可获得 5,000 个免费额度](https://docs.brightdata.com/general/account/billing-and-pricing/free-tier)。ChatGPT 爬虫工具按记录计费，每个答案对应一条记录。[定价](https://www.bright.cn/pricing/web-scraper)中列出了费率。

## 支持

此仓库中的错误：[提交 Issue](https://github.com/bright-cn/chatgpt-scraper-node/issues)，[CONTRIBUTING.md](CONTRIBUTING.md) 说明了需要提供哪些信息。有关 API、你的账户或额度的任何问题，请联系 [Bright Data 支持团队](https://brightdata.zendesk.com/hc/en-us/requests/new)。

## 许可证

[Apache 2.0](LICENSE)。

## 适用于 Python 的 ChatGPT 爬虫工具

同一款 ChatGPT 爬虫工具也提供 Python 版本：[bright-cn/chatgpt-scraper](https://github.com/bright-cn/chatgpt-scraper)。

## Bright Data 的更多开源爬虫工具

其他以相同方式构建并提供实时验证示例的 Node.js 爬虫工具：

- [适用于 Node.js 的 URL 转 Markdown（Fetch API）](https://github.com/bright-cn/fetch-api-scraper-node)
- [适用于 Node.js 的 YouTube 爬虫工具](https://github.com/bright-cn/youtube-scraper-node)
- [适用于 Node.js 的 LinkedIn 爬虫工具](https://github.com/bright-cn/linkedin-scraper-node)

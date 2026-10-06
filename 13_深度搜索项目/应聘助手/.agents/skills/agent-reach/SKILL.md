---
name: agent-reach
description: >
  Use the internet: search, read, and interact with 13+ platforms including
  Twitter/X, Reddit, YouTube, GitHub, Bilibili, XiaoHongShu (小红书), Douyin (抖音),
  WeChat Articles (微信公众号), LinkedIn, Boss直聘, RSS, Exa web search, and any web page.
  Use when: (1) user asks to search or read any of these platforms,
  (2) user shares a URL from any supported platform,
  (3) user asks to search the web, find information online, or research a topic,
  (4) user asks to post, comment, or interact on supported platforms,
  (5) user asks to configure or set up a platform channel.
  Triggers: "搜推特", "搜小红书", "看视频", "搜一下", "上网搜", "帮我查", "全网搜索",
  "search twitter", "read tweet", "youtube transcript", "search reddit",
  "read this link", "看这个链接", "B站", "bilibili", "抖音视频",
  "微信文章", "公众号", "LinkedIn", "GitHub issue", "RSS",
  "search online", "web search", "find information", "research",
  "帮我配", "configure twitter", "configure proxy", "帮我安装".
---

# Agent Reach — Usage Guide

Upstream tools for 13+ platforms. Call them directly.

Run `source ~/.agent-reach/venv/bin/activate && agent-reach doctor` to check which channels are available.

## ⚠️ Workspace & Environment Rules

**ALWAYS run commands inside the Agent Reach virtual environment.**
Before executing any CLI tool (`agent-reach`, `mcporter`, `xreach`, `yt-dlp`, python scripts), you MUST activate the environment:
`source ~/.agent-reach/venv/bin/activate` or chain it like `source ~/.agent-reach/venv/bin/activate && <command>`.

**Never create files in the agent workspace.** Use `/tmp/` for temporary output and `~/.agent-reach/` for persistent data.

## Web — Any URL

```bash
curl -s "https://r.jina.ai/URL"
```

## Web Search (Exa)

```bash
source ~/.agent-reach/venv/bin/activate && mcporter call 'exa.web_search_exa(query: "query", numResults: 5)'
source ~/.agent-reach/venv/bin/activate && mcporter call 'exa.get_code_context_exa(query: "code question", tokensNum: 3000)'
```

## Twitter/X (xreach)

```bash
source ~/.agent-reach/venv/bin/activate && xreach search "query" -n 10 --json          # search
source ~/.agent-reach/venv/bin/activate && xreach tweet URL_OR_ID --json                # read tweet (supports /status/ and /article/ URLs)
source ~/.agent-reach/venv/bin/activate && xreach tweets @username -n 20 --json         # user timeline
source ~/.agent-reach/venv/bin/activate && xreach thread URL_OR_ID --json               # full thread
```

## YouTube (yt-dlp)

```bash
source ~/.agent-reach/venv/bin/activate && yt-dlp --dump-json "URL"                     # video metadata
source ~/.agent-reach/venv/bin/activate && yt-dlp --write-sub --write-auto-sub --sub-lang "zh-Hans,zh,en" --skip-download -o "/tmp/%(id)s" "URL"
                                             # download subtitles, then read the .vtt file
source ~/.agent-reach/venv/bin/activate && yt-dlp --dump-json "ytsearch5:query"         # search
```

## Bilibili (yt-dlp)

```bash
source ~/.agent-reach/venv/bin/activate && yt-dlp --dump-json "https://www.bilibili.com/video/BVxxx"
source ~/.agent-reach/venv/bin/activate && yt-dlp --write-sub --write-auto-sub --sub-lang "zh-Hans,zh,en" --convert-subs vtt --skip-download -o "/tmp/%(id)s" "URL"
```

> Server IPs may get 412. Use `--cookies-from-browser chrome` or configure proxy.

## Reddit

```bash
curl -s "https://www.reddit.com/r/SUBREDDIT/hot.json?limit=10" -H "User-Agent: agent-reach/1.0"
curl -s "https://www.reddit.com/search.json?q=QUERY&limit=10" -H "User-Agent: agent-reach/1.0"
```

> Server IPs may get 403. Search via Exa instead, or configure proxy.

## GitHub (gh CLI)

```bash
gh search repos "query" --sort stars --limit 10
gh repo view owner/repo
gh search code "query" --language python
gh issue list -R owner/repo --state open
gh issue view 123 -R owner/repo
```

## 小红书 / XiaoHongShu (mcporter)

```bash
source ~/.agent-reach/venv/bin/activate && mcporter call 'xiaohongshu.search_feeds(keyword: "query")'
source ~/.agent-reach/venv/bin/activate && mcporter call 'xiaohongshu.get_feed_detail(feed_id: "xxx", xsec_token: "yyy")'
source ~/.agent-reach/venv/bin/activate && mcporter call 'xiaohongshu.get_feed_detail(feed_id: "xxx", xsec_token: "yyy", load_all_comments: true)'
source ~/.agent-reach/venv/bin/activate && mcporter call 'xiaohongshu.publish_content(title: "标题", content: "正文", images: ["/path/img.jpg"], tags: ["tag"])'
```

> Requires login. Use Cookie-Editor to import cookies.

## 抖音 / Douyin (mcporter)

```bash
source ~/.agent-reach/venv/bin/activate && mcporter call 'douyin.parse_douyin_video_info(share_link: "https://v.douyin.com/xxx/")'
source ~/.agent-reach/venv/bin/activate && mcporter call 'douyin.get_douyin_download_link(share_link: "https://v.douyin.com/xxx/")'
```

> No login needed.

## 微信公众号 / WeChat Articles

**Search** (miku_ai):
```python
source ~/.agent-reach/venv/bin/activate && python3 -c "
import asyncio
from miku_ai import get_wexin_article
async def s():
    for a in await get_wexin_article('query', 5):
        print(f'{a[\"title\"]} | {a[\"url\"]}')
asyncio.run(s())
"
```

**Read** (Camoufox — bypasses WeChat anti-bot):
```bash
cd ~/.agent-reach/tools/wechat-article-for-ai && source ~/.agent-reach/venv/bin/activate && python3 main.py "https://mp.weixin.qq.com/s/ARTICLE_ID"
```

> WeChat articles cannot be read with Jina Reader or curl. Must use Camoufox.

## LinkedIn (mcporter)

```bash
source ~/.agent-reach/venv/bin/activate && mcporter call 'linkedin.get_person_profile(linkedin_url: "https://linkedin.com/in/username")'
source ~/.agent-reach/venv/bin/activate && mcporter call 'linkedin.search_people(keyword: "AI engineer", limit: 10)'
```

Fallback: `curl -s "https://r.jina.ai/https://linkedin.com/in/username"`

## Boss直聘 (mcporter)

```bash
source ~/.agent-reach/venv/bin/activate && mcporter call 'bosszhipin.get_recommend_jobs_tool(page: 1)'
source ~/.agent-reach/venv/bin/activate && mcporter call 'bosszhipin.search_jobs_tool(keyword: "Python", city: "北京")'
```

Fallback: `curl -s "https://r.jina.ai/https://www.zhipin.com/job_detail/xxx"`

## RSS

```python
source ~/.agent-reach/venv/bin/activate && python3 -c "
import feedparser
for e in feedparser.parse('FEED_URL').entries[:5]:
    print(f'{e.title} — {e.link}')
"
```

## Troubleshooting

- **Channel not working?** Run `source ~/.agent-reach/venv/bin/activate && agent-reach doctor` — shows status and fix instructions.
- **Twitter fetch failed?** Ensure `undici` is installed: `npm install -g undici`. Configure proxy: `source ~/.agent-reach/venv/bin/activate && agent-reach configure proxy URL`.

## Setting Up a Channel ("帮我配 XXX")

If a channel needs setup (cookies, Docker, etc.), fetch the install guide:
https://raw.githubusercontent.com/Panniantong/agent-reach/main/docs/install.md

User only provides cookies. Everything else is your job.
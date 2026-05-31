---
name: wechat-to-obsidian
description: Download WeChat public account (公众号) articles and save to Obsidian vault. Use when the user wants to save a 公众号文章, WeChat article, or 微信文章 to their vault. Triggered by: 下载公众号文章, 保存这篇文章到obsidian, 收藏公众号, wechat article download.
---

# WeChat Article → Obsidian

Download a WeChat public account (公众号) article and save it as a clean Markdown file in the user's Obsidian vault. With comment capture. No blogger name needed — auto-detected.

## Prerequisites

- Python with `playwright` installed
- Chromium browser: `playwright install chromium`
- Script: write Appendix A to `/tmp/wx2obsidian.py` if it doesn't exist

## Steps

### 1. Get the URL

The user can just paste the URL. No other info required. Blogger name auto-detected.

### 2. Ensure dependencies

```bash
# One-time setup (skip if already done)
python3 -m venv /tmp/wx_venv && source /tmp/wx_venv/bin/activate
pip install playwright && playwright install chromium
```

### 3. Write script if needed

Check if `/tmp/wx2obsidian.py` exists. If not, write the script from Appendix A.

### 4. Run the script

```bash
source /tmp/wx_venv/bin/activate && python3 /tmp/wx2obsidian.py "ARTICLE_URL" \
  --vault "VAULT_ROOT_PATH"
```

Options:
- `--blogger "NAME"` — override auto-detected blogger name
- `--no-comments` — skip comment capture

### 5. Report results

Show the user:
- Article title + author (auto-detected)
- Blogger/folder name (auto-detected or specified)
- File path (relative to vault)
- Character count + comment count

## Features

- ✅ Full article text + images as Markdown
- ✅ **Comments section** — author names, content, likes, timestamps, author replies
- ✅ YAML frontmatter (title, author, date, blogger, source URL, tags)
- ✅ Auto-detect blogger name from page metadata (--blogger optional)
- ✅ Clean filename with date prefix

## Output structure

```
vault/公众号文章/<博主名>/<YYYY-MM-DD 文章标题>.md
```

Each file includes:
- YAML frontmatter
- Article title as H1
- Source info block
- Cleaned Markdown body content
- `## 💬 评论区` section with all comments and replies (if available)

## Troubleshooting

- **Link expired**: Use "permanent link" from WeChat share menu
- **No comments**: Article may not have comments enabled or no 精选评论
- **Playwright missing**: `pip install playwright && playwright install chromium`
- **Vault path**: Adjust `--vault` to your Obsidian vault root

---

## Appendix A: WeChat Article Downloader Script

```python
#!/usr/bin/env python3
"""
抓取微信公众号文章，清洗为 Markdown，保存到 Obsidian vault。
支持抓取评论区。
用法: python wx2obsidian.py "文章URL" [--blogger "博主名"] --vault VAULT_PATH [--no-comments]
"""

import argparse
import re
import sys
from datetime import datetime
from pathlib import Path

from playwright.sync_api import sync_playwright, TimeoutError as PlaywrightTimeout


def fetch_comments(page, timeout_ms: int = 8000) -> list[dict]:
    """滚动到评论区并提取评论数据。"""
    try:
        page.evaluate("window.scrollTo(0, document.body.scrollHeight)")
        page.wait_for_timeout(2000)

        click_triggers = [
            "#js_cmt_loadmore", ".load_more_comments", ".comment_show_all",
            ".more_comment_btn", "a:has-text('查看评论')", "a:has-text('查看全部')",
            "span:has-text('查看更多评论')", ".btn_comment",
        ]
        for trigger in click_triggers:
            try:
                btn = page.locator(trigger).first
                if btn.is_visible(timeout=1000):
                    btn.click()
                    page.wait_for_timeout(2000)
            except:
                pass

        page.evaluate("window.scrollTo(0, document.body.scrollHeight)")
        page.wait_for_timeout(1500)

        selectors = [
            ".comment__item", ".rich_media_area_extra .comment_item",
            "#comment_list .comment_item", ".discuss_list .discuss_item",
            ".comment-list .comment-item", "[class*='comment'] [class*='item']",
        ]

        comments_data = []
        for selector in selectors:
            try:
                page.wait_for_selector(selector, timeout=timeout_ms)
                comments_data = page.evaluate("""(sel) => {
                    const items = document.querySelectorAll(sel);
                    return Array.from(items).map(item => {
                        const author = item.querySelector(
                            '.comment__author, .comment_author, .nickname, .discuss_user_name'
                        );
                        const content = item.querySelector(
                            '.comment__content, .comment_content, .discuss_content, .discuss_message'
                        );
                        const likes = item.querySelector(
                            '.comment__like, .comment_like_count, .like_num'
                        );
                        const time = item.querySelector(
                            '.comment__time, .comment_time, .discuss_time'
                        );
                        const replies = item.querySelectorAll(
                            '.comment__reply, .reply_item, .discuss_reply_item'
                        );
                        const reply_list = Array.from(replies).map(r => {
                            const r_author = r.querySelector(
                                '.reply_author, .nickname, .comment__author'
                            );
                            const r_content = r.querySelector(
                                '.reply_content, .comment__content'
                            );
                            const r_time = r.querySelector(
                                '.reply_time, .comment__time'
                            );
                            return {
                                author: r_author ? r_author.textContent.trim() : '',
                                content: r_content ? r_content.textContent.trim() : '',
                                time: r_time ? r_time.textContent.trim() : '',
                            };
                        });
                        return {
                            author: author ? author.textContent.trim().replace(/\\s+/g, ' ') : '',
                            content: content ? content.textContent.trim().replace(/\\s+/g, ' ') : '',
                            likes: likes ? likes.textContent.trim() : '0',
                            time: time ? time.textContent.trim() : '',
                            replies: reply_list,
                        };
                    }).filter(c => c.content);
                }""", selector)
                if comments_data:
                    break
            except PlaywrightTimeout:
                continue

        return comments_data
    except Exception as e:
        print(f"  ⚠ 评论抓取异常: {e}")
        return []


def fetch_article(url: str, fetch_comments_flag: bool = True) -> dict:
    """用 Playwright 打开微信文章，提取标题、作者、日期、正文HTML、评论。"""
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        context = browser.new_context(
            user_agent=(
                "Mozilla/5.0 (iPhone; CPU iPhone OS 16_0 like Mac OS X) "
                "AppleWebKit/605.1.15 (KHTML, like Gecko) "
                "Version/16.0 Mobile/15E148 Safari/604.1"
            ),
            viewport={"width": 414, "height": 896},
        )
        page = context.new_page()

        try:
            page.goto(url, wait_until="domcontentloaded", timeout=30000)
            page.wait_for_selector("#js_content", timeout=15000)
        except PlaywrightTimeout:
            browser.close()
            raise RuntimeError(f"无法加载文章页面，链接可能已失效或需要验证: {url}")

        title = page.evaluate("""() => {
            const el = document.querySelector('#activity-name');
            return el ? el.textContent.trim() : '';
        }""")

        publish_date = page.evaluate("""() => {
            const el = document.querySelector('#publish_time');
            return el ? el.textContent.trim() : '';
        }""")

        author_name = page.evaluate("""() => {
            const el = document.querySelector('#js_name');
            return el ? el.textContent.trim() : '';
        }""")

        content_html = page.evaluate("""() => {
            const el = document.querySelector('#js_content');
            return el ? el.innerHTML : '';
        }""")

        images = page.evaluate("""() => {
            const imgs = document.querySelectorAll('#js_content img');
            return Array.from(imgs).map(img => ({
                src: img.src || img.getAttribute('data-src') || '',
                alt: img.alt || ''
            }));
        }""")

        comments = []
        if fetch_comments_flag:
            comments = fetch_comments(page)

        browser.close()

    return {
        "title": title.strip(),
        "publish_date": publish_date.strip(),
        "author_name": author_name.strip(),
        "content_html": content_html,
        "images": images,
        "comments": comments,
    }


def format_comments_markdown(comments: list[dict]) -> str:
    """将评论数据格式化为 Markdown。"""
    if not comments:
        return ""

    lines = ["\n---\n\n## 💬 评论区\n"]
    for c in comments:
        author = c.get("author", "匿名")
        time_str = c.get("time", "")
        likes = c.get("likes", "0")
        content = c.get("content", "")

        time_info = f" · {time_str}" if time_str else ""
        like_info = f" · 👍 {likes}" if likes and likes != "0" else ""

        lines.append(f"**{author}**{time_info}{like_info}")
        lines.append(f"\n> {content}\n")

        replies = c.get("replies", [])
        for r in replies:
            r_author = r.get("author", "")
            r_content = r.get("content", "")
            r_time = r.get("time", "")
            r_time_info = f" · {r_time}" if r_time else ""
            if r_author and r_content:
                lines.append(f"> 🔸 **{r_author}**{r_time_info}：{r_content}\n")

        lines.append("")

    lines.append(f"> 📊 共 {len(comments)} 条评论\n")
    return "\n".join(lines)


def html_to_markdown(html: str) -> str:
    """将微信文章 HTML 清洗为 Markdown。"""
    text = html

    text = re.sub(r'<style[^>]*>.*?</style>', '', text, flags=re.DOTALL | re.IGNORECASE)

    text = re.sub(r'<img[^>]+data-src="([^"]*)"[^>]*/?>', r'\n\n![](\1)\n\n', text)
    text = re.sub(r'<img[^>]+src="([^"]*)"[^>]*/?>', r'\n\n![](\1)\n\n', text)

    text = re.sub(r'<br\s*/?>', '\n', text)
    text = re.sub(r'<p[^>]*>', '\n\n', text)
    text = re.sub(r'</p>', '', text)

    text = re.sub(r'<strong[^>]*>', '**', text)
    text = re.sub(r'</strong>', '**', text)
    text = re.sub(r'<b[^>]*>', '**', text)
    text = re.sub(r'</b>', '**', text)

    text = re.sub(r'<em[^>]*>', '*', text)
    text = re.sub(r'</em>', '*', text)
    text = re.sub(r'<i[^>]*>', '*', text)
    text = re.sub(r'</i>', '*', text)

    for level in range(1, 4):
        text = re.sub(f'<h{level}[^>]*>', '\n\n' + '#' * level + ' ', text)
        text = re.sub(f'</h{level}>', '\n', text)

    text = re.sub(r'<section[^>]*>', '\n', text)
    text = re.sub(r'</section>', '\n', text)
    text = re.sub(r'<span[^>]*>', '', text)
    text = re.sub(r'</span>', '', text)

    text = re.sub(r'<blockquote[^>]*>', '\n> ', text)
    text = re.sub(r'</blockquote>', '\n', text)

    text = re.sub(r'<[^>]+>', '', text)

    text = re.sub(r'\n{3,}', '\n\n', text)
    text = re.sub(r' +\n', '\n', text)
    text = re.sub(r' +', ' ', text)

    lines = [line.strip() for line in text.split('\n')]
    return '\n'.join(lines).strip()


def sanitize_filename(name: str) -> str:
    name = re.sub(r'[\\/:*?"<>|]', '-', name)
    return name.strip()[:80]


def save_to_vault(content: str, metadata: dict, vault_root: str, blogger: str):
    author_dir = sanitize_filename(blogger)
    out_dir = Path(vault_root) / "公众号文章" / author_dir
    out_dir.mkdir(parents=True, exist_ok=True)

    title = metadata.get("title", "未命名")
    date_str = metadata.get("publish_date", datetime.now().strftime("%Y-%m-%d"))
    date_match = re.search(r'(\d{4})[年-](\d{1,2})[月-](\d{1,2})', date_str)
    if date_match:
        date_str = f"{date_match.group(1)}-{date_match.group(2).zfill(2)}-{date_match.group(3).zfill(2)}"
    else:
        date_str = datetime.now().strftime("%Y-%m-%d")

    filename = f"{date_str} {sanitize_filename(title)}.md"
    filepath = out_dir / filename

    frontmatter = f"""---
title: "{title}"
author: "{metadata.get('author_name', blogger)}"
date: {date_str}
blogger: "{blogger}"
source: "{metadata.get('url', '')}"
tags:
  - 公众号文章
  - {blogger}
---

# {title}

> 来源：{blogger} · 微信公众号
> 日期：{date_str}
> 原文链接：{metadata.get('url', '')}

---

{content}
"""

    filepath.write_text(frontmatter, encoding='utf-8')
    return filepath


def main():
    parser = argparse.ArgumentParser(description="微信公众号文章 → Obsidian Markdown")
    parser.add_argument("url", help="微信公众号文章链接")
    parser.add_argument("--blogger", "-b", default=None, help="博主名称（可选，不传则自动识别）")
    parser.add_argument("--vault", "-v", required=True, help="Obsidian vault 根目录")
    parser.add_argument("--no-comments", action="store_true", help="跳过评论抓取")
    args = parser.parse_args()

    vault_path = Path(args.vault)
    if not vault_path.exists():
        print(f"❌ Vault 目录不存在: {vault_path}", file=sys.stderr)
        sys.exit(1)

    fetch_comments_flag = not args.no_comments

    print("🌐 正在抓取文章...")
    try:
        article = fetch_article(args.url, fetch_comments_flag)
    except RuntimeError as e:
        print(f"❌ {e}", file=sys.stderr)
        sys.exit(1)

    title = article["title"]
    if not title:
        print("❌ 未能提取文章标题", file=sys.stderr)
        sys.exit(1)

    print(f"📄 标题: {title}")
    print(f"✍️  作者: {article['author_name']}")
    print(f"📅 日期: {article['publish_date']}")

    blogger = args.blogger if args.blogger else article["author_name"]
    if not blogger:
        blogger = "未分类"
    print(f"📂 博主: {blogger}{' (自动识别)' if not args.blogger else ''}")

    comments = article.get("comments", [])
    if fetch_comments_flag:
        if comments:
            print(f"💬 抓到 {len(comments)} 条评论")
        else:
            print("💬 本文无评论或评论未开放")

    print("🧹 清洗为 Markdown...")
    markdown = html_to_markdown(article["content_html"])

    if comments:
        markdown += "\n" + format_comments_markdown(comments)

    metadata = {
        "title": title,
        "publish_date": article["publish_date"],
        "author_name": article["author_name"],
        "url": args.url,
    }

    print("💾 保存到 vault...")
    filepath = save_to_vault(markdown, metadata, str(vault_path), blogger)

    print(f"\n✅ 完成!")
    print(f"📁 {filepath}")
    print(f"📊 文章长度: {len(markdown)} 字符")
    if comments:
        print(f"💬 评论数: {len(comments)} 条")


if __name__ == "__main__":
    main()
```

# tical-content — AI-Generated Content Engine

> **Philosophy:** Don't fight anti-crawl. Make content worth crawling.

## The Problem
Search engines block our VPS IPs when we try to scrape them for data. Fighting anti-crawl is an arms race we can't win.

## The Solution
Instead of scraping others, **generate valuable content that search engines will naturally crawl and index**.

We already produce:
- Multi-AI code reviews (DeepSeek+Gemini+Grok)
- tical-chat human↔AI dialogues
- Architecture decisions and planning docs
- System analysis reports

Pipeline:
```
AI output → sanitize (strip IPs/keys) → format as blog post → publish to public site → SEO traffic
```

## Implementation Steps (for 扣子)

### Step 1: Content Sanitizer
Script that strips sensitive data before publishing (IPs, tokens, keys, personal info).

### Step 2: Blog Templates
- Tech blog: code review findings, architecture decisions
- Dialogue digest: interesting AI conversations
- Analysis reports: system metrics, trends

### Step 3: Auto-Publish
Sanitize → format → git commit → push to GitHub Pages or public site.

## Priority
- P0: Sanitizer + first blog post template
- P1: Dialogue digest template
- P2: Auto-publish workflow

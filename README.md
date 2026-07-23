# Daily Briefing

Automated daily audio news briefing, generated every morning at 6:00 AM (Europe/Paris) by a scheduled Claude Code cloud routine — no local machine or server involved.

## What's in this repo

- `audio/YYYY-MM-DD.mp3` — the day's ~25-minute spoken briefing (French), generated with edge-tts
- `notes/YYYY-MM-DD.md` — the full script as text, with sources linked inline, for going deeper on any topic
- `feed.xml` — the podcast RSS feed, served publicly via GitHub Pages at https://magicapsss.github.io/daily-news/feed.xml

Add that feed URL to any podcast app (AntennaPod, Podcast Addict, Pocket Casts, etc.) to get new episodes automatically each morning.

## Retention

Audio files older than 30 days are pruned automatically to keep the repo small. Notes in `notes/` are kept indefinitely as a permanent archive.

## How it's generated

There's no script in this repo. Each morning, a fresh cloud agent session researches the day's news, writes the script, converts it to speech, updates the feed, and pushes — driven by the routine's stored prompt, not by code committed here.

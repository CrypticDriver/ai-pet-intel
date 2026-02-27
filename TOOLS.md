# TOOLS.md - Local Notes

Skills define _how_ tools work. This file is for _your_ specifics — the stuff that's unique to your setup.

## What Goes Here

Things like:

- Camera names and locations
- SSH hosts and aliases
- Preferred voices for TTS
- Speaker/room names
- Device nicknames
- Anything environment-specific

## Examples

```markdown
### Cameras

- living-room → Main area, 180° wide angle
- front-door → Entrance, motion-triggered

### SSH

- home-server → 192.168.1.100, user: admin

### TTS

- Preferred voice: "Nova" (warm, slightly British)
- Default speaker: Kitchen HomePod
```

## Why Separate?

Skills are shared. Your setup is yours. Keeping them apart means you can update skills without losing your notes, and share skills without leaking your infrastructure.

---

Add whatever helps you do your job. This is your cheat sheet.

## Credentials

### GitHub

- **Token 路径**: `/home/ubuntu/.openclaw/workspace/.credentials/github.txt`
- **类型**: Classic Token (共享，所有 agent 可用)
- **用途**: 提交调研报告、更新竞品分析、推送市场数据
- **读取方式**: `cat /home/ubuntu/.openclaw/workspace/.credentials/github.txt`

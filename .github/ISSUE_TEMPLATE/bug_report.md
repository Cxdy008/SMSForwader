---
name: Bug report
about: Report a bug in SMSForwarder
title: "[Bug] "
labels: bug
assignees: ""
---

> **Privacy:** SMS often contain one-time codes, bank alerts and private messages. Before posting anything, redact SMS text, phone numbers, the bot token and the chat ID from descriptions, logs and screenshots.

## Description

A clear and concise description of the bug.

## Environment

- **Android version:** (e.g., Android 15, API 35)
- **Device / manufacturer:** (e.g., Xiaomi Redmi Note 13, HyperOS 2)
- **App version / commit:**
- **Battery optimization exempted:** yes / no
- **Manufacturer autostart enabled (if applicable):** yes / no / not applicable

## Reproduction Steps

1.
2.
3.

## Expected Behavior

What should have happened (e.g., "the SMS arrives in Telegram").

## Actual Behavior

What happened instead. Was any SMS **lost** (never forwarded), **duplicated**, or only **delayed**?

## Logs

```
Paste relevant `adb logcat` output here (redacted)
```

If you see SMS text, a phone number or the token in the logs, that is a bug in itself: see the safety contract in `docs/pipeline.md`.

## Checklist

- [ ] I have redacted SMS content, phone numbers, the bot token and the chat ID
- [ ] I have searched existing issues for duplicates
- [ ] I am on the latest `main`

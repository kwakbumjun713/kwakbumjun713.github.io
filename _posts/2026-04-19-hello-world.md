---
title: "Hello, World"
date: 2026-04-19 12:00:00 +0900
categories: [misc]
tags: [meta]
---

블로그 첫 포스트. 앞으로 여기에 CTF writeup, 취약점 분석, 삽질기를 기록한다.

## 코드 블록 테스트

```python
def pwn():
    payload = b"A" * 72 + p64(0xdeadbeef)
    print(payload)
```

## 인용 테스트

> "어떤 시스템이든 결국 깨진다 — 시간 문제일 뿐."

끝.

---
name: ghibli-recipe
description: Use when the user wants a quick (usually 15-minute) recipe written as a Markdown file with a Studio-Ghibli-style thumbnail image and a matching PDF, optionally committed and pushed to GitHub — e.g. "15분 레시피 하나 만들어줘", "지브리풍 썸네일 넣어서 레시피 md 만들어줘", "레시피 pdf로 만들어서 push해줘".
---

# Ghibli Recipe Workflow

주어진 요리(또는 사용자가 정한 주제)에 대해 15분 레시피 Markdown → 지브리풍 나노바나나(Gemini 이미지 생성) 썸네일 → PDF → (요청 시) git commit/push 까지 만드는 반복 워크플로우.

## 0. 사전 확인

- 대상 폴더: 사용자가 지정하지 않으면 기존 `week-N` 폴더 중 가장 최근 것을 사용하거나, 없으면 새로 만든다. 폴더 구조는 `Glob "week-*/**"`로 먼저 확인한다.
- 파일명: 요리 이름을 그대로 파일명으로 쓴다 (예: `참치마요덮밥.md`).
- **API 키**: 나노바나나(Gemini) API 키를 코드나 파일에 절대 하드코딩하지 않는다. `$env:GEMINI_API_KEY` 환경변수를 먼저 확인하고, 없으면 사용자에게 요청한다. 사용자가 채팅에 키를 직접 붙여넣으면, 사용은 하되 **작업이 끝나면 반드시 키를 재발급하라고 안내**한다 — 채팅 로그에 노출된 키는 이미 유출된 것으로 간주한다.

## 1. 레시피 Markdown 작성

아래 템플릿을 따른다. 조리 순서 각 단계에 소요 시간을 표기하고 합이 총 소요 시간(보통 15분)과 맞아야 한다.

```markdown
# {요리명} ({N}분 완성)

![{요리명} 썸네일]({썸네일파일명}.png)

{한 줄 소개}

## 재료 ({인분})

- 재료1
- 재료2

## 조리 순서

1. **{단계명} ({분})** — 설명
2. ...

## 꿀팁

- 팁1
- 팁2

**총 소요 시간: 약 {N}분**

## 지브리풍 썸네일 프롬프트 (외부 이미지 생성기용)

> {아래 2단계에서 쓴 프롬프트와 동일한 내용}
```

## 2. 나노바나나로 지브리풍 썸네일 생성

Gemini `gemini-2.5-flash-image` 모델(코드네임 "나노바나나")을 PowerShell `Invoke-RestMethod`로 직접 호출한다. 프롬프트는 요리에 맞게 매번 새로 작성하되, 다음 톤을 유지한다:

> Studio Ghibli style illustration, warm hand-painted anime background, a cozy wooden table with soft golden afternoon light, {요리 묘사 — 그릇/컵 형태, 토핑, 데코 소품 1개}, gentle steam wisps rising (해당 시), whimsical and nostalgic mood, soft brush strokes, warm pastel color palette, no text, 4:3 aspect ratio

```powershell
$ErrorActionPreference = 'Stop'
$apiKey = $env:GEMINI_API_KEY   # 하드코딩 금지 — 없으면 사용자에게 요청
$prompt = "..."  # 위 톤을 따르는 프롬프트

$body = @{
    contents = @(@{ parts = @(@{ text = $prompt }) })
} | ConvertTo-Json -Depth 10

$uri = "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-image:generateContent?key=$apiKey"

$response = Invoke-RestMethod -Uri $uri -Method Post -ContentType 'application/json' -Body $body

# 주의: parts[0]이 항상 이미지는 아니다. text 파트가 먼저 올 수 있으므로
# inlineData를 가진 파트를 명시적으로 찾아야 한다.
$imgPart = $response.candidates[0].content.parts | Where-Object { $_.inlineData } | Select-Object -First 1
$bytes = [System.Convert]::FromBase64String($imgPart.inlineData.data)
[System.IO.File]::WriteAllBytes("{week 폴더}\thumbnail_{요리}.png", $bytes)
```

생성 후 `Read` 도구로 PNG를 열어 이미지가 레시피와 맞는지 확인한다. md 파일의 이미지 링크를 이 PNG로 연결한다.

## 3. PDF로 변환

이 환경에는 pandoc/wkhtmltopdf가 없다. 대신 **Edge 헤드리스 print-to-pdf**를 쓴다.

1. 레시피 내용을 담은 HTML을 스크래치패드에 작성한다 (제목/재료/순서/꿀팁/프롬프트 섹션, 요리 톤에 맞는 accent color). `<img>` 태그의 `src`는 **절대 file:// 경로**로 지정한다 (상대경로는 로드 실패 원인이 됨):
   ```html
   <img src="file:///c:/Users/pc/Desktop/tuesdayclass/week-1/thumbnail_xxx.png" alt="...">
   ```
2. Edge 헤드리스로 변환. **반드시 `--user-data-dir`을 스크래치패드 하위의 전용 폴더로 지정한다** — 지정하지 않으면 이전 헤드리스 실행이 남긴 프로세스/프로필 락 때문에 다음 실행이 (에러 없이) 조용히 PDF를 만들지 않고 종료하는 경우가 있다:
   ```powershell
   $edge = "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe"
   & $edge --headless --disable-gpu --no-sandbox `
       --user-data-dir="{scratchpad}\edge-profile" `
       --print-to-pdf="{week 폴더}\{요리명}.pdf" `
       --print-to-pdf-no-header `
       "file:///{scratchpad html 절대경로}"
   Start-Sleep -Seconds 2
   Test-Path "{week 폴더}\{요리명}.pdf"   # 반드시 확인 — False면 아래 참고
   ```
   실패 시(`Test-Path`가 False): `Get-Process msedge | Stop-Process -Force`로 남은 프로세스를 정리한 뒤 재시도한다.

   stderr에 찍히는 Chromium `ERROR:` 로그(task_manager, usb_descriptors 등)는 무해하니 무시한다. **`2>$null`로 native exe의 stderr를 리다이렉트하지 말 것** — PowerShell 5.1에서 정상 종료도 실패로 잘못 판정되는 경우가 있다.
3. `Read` 도구로 결과 PDF를 열어 이미지와 한글 텍스트가 깨지지 않고 들어갔는지 확인한다.

## 4. Git commit & push (사용자가 요청한 경우에만)

- `git`이 PATH에 없을 수 있다 — 필요하면 `$env:Path += ";C:\Program Files\Git\bin"`.
- **이번 작업과 무관한 파일은 절대 스테이징하지 않는다.** `git status`로 확인 후, 방금 만들거나 수정한 레시피 md/png/pdf만 명시적으로 `git add`한다. `.claude/`나 관련 없는 루트 파일은 건드리지 않는다.
- 커밋 메시지는 이번에 추가/변경한 레시피를 간단히 요약한다.
- push는 `git push origin main` (또는 현재 브랜치) — 원격에 반영되는 되돌리기 어려운 동작이므로, 사용자가 명시적으로 push까지 요청했을 때만 실행한다.

## 체크리스트

- [ ] md 조리 단계 시간 합 = 총 소요 시간
- [ ] 썸네일 PNG가 실제 요리 설명과 일치 (Read로 육안 확인)
- [ ] PDF에 이미지 + 한글 텍스트 정상 렌더링 확인
- [ ] API 키를 파일/커밋에 남기지 않았는지 확인
- [ ] git add 범위가 이번 작업 파일로 한정됐는지 확인

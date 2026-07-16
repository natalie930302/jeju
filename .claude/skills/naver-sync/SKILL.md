---
name: naver-sync
description: Use this skill when the user asks to update/verify/sync locations in this Jeju itinerary (index.html) against Naver Map — coordinates, hours, price, parking, menu, or note formatting. Also use it whenever adding a NEW location that has a naver_url, or reformatting existing notes into the 4-part template. Triggers on phrases like "更新 Naver 資料", "抓 Naver", "座標對不對", "筆記格式", "加一個新地點".
---

# Naver Place sync for index.html itinerary

This repo has no dedicated browser tool and `WebFetch` is blocked for the entire `naver.com` domain (tested — always 403s, even `www.naver.com`). The only way to read a Naver Place page is a real headless browser via Bash/Playwright. This skill documents the working method so it doesn't need to be re-derived.

## 0. Data model

All location data lives in **one minified JSON array** on a single line inside `index.html`:
```js
const DATA=[{...},{...}];
```
Find the line number once per session:
```bash
grep -n '^const DATA=' index.html   # currently line 310 (1-indexed) / index 309 (0-indexed)
```
Never hand-edit that line. Always: extract → parse with Node → mutate as JS objects → `JSON.stringify` → splice back into the same line. Keep a working `data.json` (pretty-printed) in the repo root while iterating, but **delete it before committing** — it is scratch, not tracked.

Round-trip pattern:
```js
// extract
const fs = require('fs');
const html = fs.readFileSync('index.html', 'utf-8');
const lines = html.split('\n');
const data = JSON.parse(lines[309].trim().replace(/^const DATA=/, '').replace(/;$/, ''));
fs.writeFileSync('data.json', JSON.stringify(data, null, 2));

// ...edit data.json / mutate `data` via scripts...

// write back
lines[309] = 'const DATA=' + JSON.stringify(data) + ';';
fs.writeFileSync('index.html', lines.join('\n'));
```

Field schema per item: `cat, route, zh, en, kr, naver_url, addr_en, addr_kr, lat, lng, hours, price, dish?, day, note, place_id?`.

## 1. One-time environment setup (per machine, not per session)

Playwright is not a project dependency (would bloat the repo) — install it in the **scratchpad**, not in the repo:
```bash
mkdir -p "$SCRATCHPAD/naverscrape" && cd "$SCRATCHPAD/naverscrape"
npm init -y && npm install playwright
npx playwright install chromium   # downloads ~150MB, one-time per machine
```
If chromium is already installed (check `~/AppData/Local/ms-playwright/`), skip straight to scraping.

## 2. Scraping a single Naver place

The generic URL works for **every category** (restaurant, cafe, attraction, shop, accommodation — no need to guess `/restaurant/` vs `/cafe/` vs `/place/` path segments):
```
https://pcmap.place.naver.com/place/{placeId}/home
```
`placeId` is the numeric id at the end of the stored `naver_url` (`.../place/1042054066` → `1042054066`).

The page embeds a full Apollo GraphQL cache in `window.__APOLLO_STATE__` after `networkidle` — this has everything, no need to click through tabs (메뉴/정보/리뷰) or fight a captcha widget (there's a dormant `ncaptcha-iframe` in the DOM but it never blocks this data).

```js
const { chromium } = require('playwright');
const browser = await chromium.launch({ headless: true });
const page = await browser.newPage({
  userAgent: 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
});
await page.goto(`https://pcmap.place.naver.com/place/${placeId}/home`, { waitUntil: 'networkidle', timeout: 25000 });
await page.waitForTimeout(1200);
const state = await page.evaluate(() => window.__APOLLO_STATE__ || null);
```

### Where each fact lives inside `state`

- `state['PlaceDetailBase:' + placeId]` → `name, category, coordinate:{x(lng),y(lat)}, conveniences[] (편의 tags, e.g. "주차"), visitorReviewsTotal, visitorReviewsScore, roadAddress, address`
- `state.ROOT_QUERY['placeDetail({"input":{"deviceType":"pcmap","id":"'+placeId+'","isNx":false}})']` → the big nested object (call it `pd`). Its keys are parameterized (Apollo field-arg suffixes), find them by prefix match since exact args can vary:
  - `pd['newBusinessHours({"format":"restaurant"})']` → array of `NewBusinessHour`; each has `businessHours[]` (per day-group: `day` ("매일" or "월"/"화"/... or "월(7/20)" for a specific date), `businessHours:{start,end}`, `breakHours[]`, `lastOrderTimes[]`), `comingRegularClosedDays` (e.g. "매달 2, 4번째 화요일 정기 휴무" — **read this, don't assume weekly patterns from a single week's snapshot**), `freeText`.
  - `pd['informationTab({"providerSource":["pbp"]})']` → `parkingInfo:{description, basicParking:{isFree, normalFeeDescription, extraFeeDescription}}`, `keywordList[]` (review tag cloud), `facilities[]` (resolve `{__ref}` pointers back into `state`), `pet`.
  - `pd['menus({"source":["tpirates"]})']` isn't reliable; instead scan **top-level** `state` keys matching `` `Menu:${placeId}_${n}` `` → each has `name, price, recommend:boolean, description`. `recommend:true` items are Naver's own picks — prefer these for the 推薦 section over guessing.
- A day-group with `businessHours: null` and `description: "정기휴무 (매주 수요일)"` means closed that day — don't skip it, it's the closure info.
- Ticket-tiered places (attractions) list prices as menu items too, e.g. `성인:19000`, `청소년:16000` — use the adult (성인/어른/일반) tier only, per this project's pricing rule.

### Batch scraping

For >5 places, script a loop with 1–2s jittered delay between requests and 2 retry attempts (occasional `no_apollo_state` timeouts happen, ~3% of requests); write results incrementally to a JSON file keyed by placeId so a killed/timed-out run can resume (skip ids already present without an `.error`). Run via Bash `run_in_background` if the batch will exceed ~8 minutes — the 10-minute tool timeout does not stop a background process from continuing, and you can just re-invoke the same command to pick up where it left off.

## 3. Note format (confirmed template — do not deviate without asking)

Every `note` is 2–4 lines joined by `\n`, each line a distinct labelled section — **no other section labels, no per-sentence line breaks**:

```
【分類】介紹句，涵蓋環境氛圍、特色賣點；若景點內有子店家/附設店（即使沒有獨立 Naver 標記）也在這裡提到。
推薦：中文名稱(韓文原名)—說明＋價格；可多項，用「；」分隔在同一行內，不要逐項換行。同名項目間可省略「中文名稱(韓文原名)」的韓文部分重複。
停車：停車資訊。沒有資料就寫「無專屬停車場，可停ＸＸ」或省略此行（純自然景觀類常常沒有）。
注意事項：服務態度、售罄風險、公休異動、評論實際內容摘要（不是「評論N則，關鍵字：X、Y」這種列表式寫法——要用自然語言描述評論在說什麼）等。沒有內容就整行省略，不要留空段落。
```

Rules learned from doing this across ~175 entries:
- **Not every line is mandatory.** Airports/rental services skip 推薦; free scenic spots often skip both 推薦 and 停車 if there's nothing to say. Never pad a line with filler just to fill the template.
- **Preserve, don't summarize.** When restructuring an existing rich note (e.g. a market with a dozen vendor call-outs), keep every vendor/item under 推薦 in one long semicolon-joined line — do not cut down to "2–3 items" if the source already had more real, specific value. The 2–3-item guidance is a floor for thin places, not a ceiling for rich ones.
- **Trip.com bookings**: if `price` already shows a Trip.com summary (see §4), do not also put the booking URL / "比分開購買省NT$X" detail in the note — it's redundant with the price pill. Drop it entirely when reformatting.
- **No phone numbers** in notes (user preference — official websites are fine to keep).
- Cross-check every fact in the old note against the freshly scraped Apollo data before keeping it — hours, closed-days, and prices in older notes have been wrong often enough (~15% of checked entries) that "looks plausible" is not enough. Silently-wrong closed-days (e.g. a note claiming "週六公休" when Naver's live schedule shows Monday is the actual closed day) are the single most common and highest-impact error class found.

## 4. `price` field format (exactly one of these four shapes — no exceptions)

| Case | Format | Example |
|---|---|---|
| Restaurant/cafe per-person estimate | `約₩X,000-Y,000/人` (or exact `₩X,000/人`) | `約₩30,000-50,000/人` |
| Attraction bookable via Trip.com | `Trip.com預訂NT$X起` | `Trip.com預訂NT$388起` |
| Attraction, on-site ticket only | `門票現場₩X/人` | `門票現場₩5,000/人` |
| Free | `免費` (add `（OO另計）` suffix if partially free) | `免費（餐飲另計）` |
| Retail / no fixed spend | `依消費而定` (the only accepted wording — not "依商品而定"/"依各店家"/"依店家菜單") | |

Never mix Korean text, multiple named menu items, or NT$ parentheticals into this field — that detail belongs in the note's 推薦/注意事項 lines, not the price pill. Only-adult pricing (no 청소년/어린이/군인/도민 tiers). NT$ conversion rate for this project: **1 TWD ≈ 46.7 KRW** (round to nearest 5 NT$).

## 5. Validation before committing

```bash
node -e "
const fs=require('fs');
const lines=fs.readFileSync('index.html','utf-8').split('\n');
const data=JSON.parse(lines[309].trim().replace(/^const DATA=/,'').replace(/;$/,''));
console.log('items:', data.length, 'bad coords:', data.filter(d=>typeof d.lat!=='number'||typeof d.lng!=='number').length);
"
```
Then actually render it with the same Playwright browser (not just JSON-parse) and confirm marker count + zero console errors:
```js
await page.goto('file:///' + process.cwd().replace(/\\/g,'/') + '/index.html', { waitUntil: 'networkidle' });
document.querySelectorAll('.leaflet-marker-icon').length  // should equal data.length
```
Delete `data.json` and any scratch `apply_*.js` files from the repo root before `git add` — only `index.html` should be staged.

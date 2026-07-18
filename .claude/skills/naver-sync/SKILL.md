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

Field schema per item: `cat, route, zh, en, kr, naver_url, addr_en, addr_kr, lat, lng, hours, price, dish?, day, note, place_id?, kakao_id?, hot?`.

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
- **The structured `Menu:` entities are not the full picture.** Small/informal items (seasonal specials, packaged retail add-ons like gift-box sweets) are often only shown as a photographed menu board or a plain product photo, never entered as a formal menu row. If you're about to write a hedge like "未能核實是否仍販售" for something the source material (a screenshot, a review) claims exists, check the photo gallery before hedging: `page.evaluate(() => Array.from(document.querySelectorAll('img')).map(img => ({src: img.src, alt: img.alt})))` on the `/place/{id}/menu` route — look for `alt` starting with `메뉴판` (menu board), and for generic `business`-type photos in `PlaceDetailTopPhotoItem:business_*` inside `__APOLLO_STATE__`, which often show product photos of exactly these off-menu items. Download and view the image directly rather than guessing from the filename.

### CRLF gotcha
After a `git add`/`commit`/merge touches `index.html`, git's `core.autocrlf` normalizes the working-tree copy to CRLF line endings, which breaks the naive `lines[idx].slice('const DATA='.length, -1)` extraction (it only strips one trailing char, leaving a stray `\r` or `;` and causing `JSON.parse` to throw "Unexpected non-whitespace character"). Always strip a trailing `\r` before slicing off the `;`:
```js
const trimmed = lines[idx].replace(/\r$/, '');
const data = JSON.parse(trimmed.slice('const DATA='.length, -1));
```

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

## 5. Cross-checking review quality (Naver keyword themes + Kakao Map rating)

**Mandatory for every new 美食/咖啡廳/購物 entry, not just the batch check that happened once.** When adding a new place in these categories, always run this section's checks before finishing — don't just pull hours/menu/price and stop. A place added without this step is missing data every other entry in these categories has, which is exactly the kind of gap a user will notice and have to ask about.

Only meaningful for actual businesses (美食/咖啡廳/購物) — beaches, islands, and other 景點 without a storefront have no rating on Kakao Map and often no theme breakdown on Naver either.

**Naver's real vote-counted review themes** (better than `keywordList`, which is just SEO tags): `state['VisitorReviewStatsResult:' + placeId].analysis.themes` → array of `{code, label, count}`, e.g. `{code:"taste", label:"맛", count:471}`. This is the actual capsule-tag ranking shown at the top of the review tab. If `taste`/`total`(만족도) dominates, the place is substance-driven; if `mood`(분위기)/photo-adjacent themes rank above taste, it's a style-over-substance signal worth a note.

**Kakao Map has no Apollo-style state** — it's plain server-rendered text, so scrape `document.body.innerText` and parse it, not CSS selectors (Kakao's class names aren't stable/guessable). Search `https://map.kakao.com/?q={query}` with the place's `kr` name. Each result block is delimited by `즐겨찾기\n로드뷰\n`; parse per block: first line (strip a leading `A`/`B`/... letter marker) is name+category, `별점` is followed by the rating then a `N건` line, OR the block instead says `후기 미제공` (no rating computed yet even if reviews exist — itself a useful "too new to trust" signal), a `리뷰 N` line gives review count, and an address line starts with the province name. **Always filter results to addresses starting with `제주`** — a bare business name like "온오프" or "해지개" returns dozens of unrelated Seoul/Incheon/Gyeongnam matches with the same name, and picking the wrong one silently reports a different store's rating. Small/local places sometimes aren't listed on Kakao at all (zero 제주 results) — that's a valid outcome, not a bug.

**The popup also has a one-click Kakao button for the user**, separate from the scraping method above: in `showPopup()` (index.html), a second button (`.pc-btn-k`, Kakao's brand yellow `#FEE500`) sits next to the existing Naver Map button, linking straight to the place's Kakao page — `https://place.map.kakao.com/{kakaoId}` when a `kakao_id` is stored on the item, falling back to a live search link (`https://map.kakao.com/?q=`+encodeURIComponent(p.kr)) when it isn't. **Prefer the direct ID link** — a search-query link makes the user click through results themselves, which defeats the point of a one-click button.

**Finding the Kakao place ID**: don't click "상세보기" (it opens a real new tab/popup, slow and flaky to drive) — the link is already a plain `<a>` in the DOM with the ID in its `href`, no click needed:
```js
await page.goto('https://map.kakao.com/?q=' + encodeURIComponent(query), { waitUntil: 'networkidle', timeout: 20000 });
const items = await page.evaluate(() => Array.from(document.querySelectorAll('a.moreview')).map(a => ({
  href: a.href,                                    // "https://place.map.kakao.com/{id}"
  liText: a.closest('li') ? a.closest('li').innerText : '',  // same block text as the search-parsing method above
})));
```
Each `<li>` (one per search result) contains both the `a.moreview` detail link and the same info block (name/rating/address) the plain-text search method parses — so **do both extractions from the same page load**, don't scrape twice. Filter to `제주`-prefixed addresses as before. When more than one 제주 result comes back for a query (common for generic names), don't just take the first — score candidates by address-token overlap against the item's stored `addr_kr` (see `pickBest()` in the scratchpad's `kakao_id_batch.js` pattern) and require a nonzero score before trusting the match; otherwise leave `kakao_id` unset rather than risk linking to the wrong branch.

Add the field as `kakao_id` (string) on items where a confident match was found — leave it absent (not `null`) when no match/ambiguous, so `p.kakao_id ?` in the popup code falls through to the search-link fallback cleanly.

**Method that doesn't work: 저장 (bookmark/save) count.** Confirmed by testing both desktop (`pcmap.place.naver.com`) and mobile (`m.place.naver.com`) — the 저장 button exists in the UI but Naver never renders the actual saved-count number on the web in either surface, only on the native app. Don't attempt this; there's no field to scrape.

**Individual review text (real 영수증 리뷰 content, not just aggregate stats)**: top-level `state` keys matching `VisitorReview:{reviewId}` (visit `/place/{id}/review/visitor?reviewSort=recent`) → `body` (the actual review text), `rating`, `votedKeywords[]` (the specific per-review capsule tags, e.g. "음식이 맛있어요"), `item` (what they ordered), and a `receiptInfoUrl` field — its presence confirms the review is receipt-verified (구매 인증), the highest-trust review type. Pull a couple of these when a place needs a real quote (e.g. to explain a Naver/Kakao score gap), not for every place — it's a second full page load on top of the stats fetch.

**A small Kakao sample is noise, not signal — don't report it as if it were.** Chain retail (Olive Young, ABC MART, Daiso, UNIQLO...) splits its real customer base across dozens of branches, so any single branch's Kakao rating is usually just 1–15 raters — essentially random. Standalone restaurants/cafes don't have this dilution, so even a modest count there (~8+) is meaningful. Use a category-aware minimum before including a Kakao score at all: **≥20 ratings for 購物, ≥8 for 美食/咖啡廳**; below that, omit Kakao from the note entirely rather than printing a misleadingly precise-looking number. The same logic applies to Naver itself in rare cases — a `VisitorReviewStatsResult.review.avgRating` under 2.0 backed by fewer than ~20 total reviews is often a single outlier rater (check `scores`/`authorCount`, not just `avgRating`) rather than a real consensus; don't report it as a plain score without checking the sample first.

Append the result as a sentence at the end of the note's 注意事項 line (create the line if the note doesn't have one — 注意事項 is always last, see §3), e.g. `Naver 4.95分(17,577則)；Kakao 4.8分(109則)，與Naver評分一致，可信度高。` Skip the sentence entirely if neither Naver nor Kakao clears its threshold — don't force a low-confidence number into the note.

### `hot` flag / ⭐ star badge in the sidebar list

A place gets `"hot": true` (renders a ⭐ before its name in the sidebar list, via `.phot` in `renderList()`) only if **all** of these hold — check in this order, don't skip the last one:

1. **Naver rating ≥ 4.7 with ≥ 20 reviews.**
2. **Not polarized**: if Kakao clears its category threshold above, the Naver−Kakao gap must be < 0.5. A place Naver rates 4.8 and Kakao rates 4.1 is a real disagreement, not noise — exclude it even though step 1 alone would pass.
3. **Floor-check by reading actual review text, not the `scores` bucket array.** The `scores` array on `VisitorReviewStatsResult.review` cannot be reliably mapped to star levels — its `score` field is always `null` and there's no visual histogram on the page to cross-reference against, so any ordering assumption (ascending/descending) is a guess. Instead fetch individual `VisitorReview` entities from `/place/{id}/review/visitor?reviewSort=recent` (recent, not the default `recommend` sort — recommend biases toward already-upvoted/positive content and undersamples complaints) and read the `body` text of ~10 of them directly. Look for a genuine, specific complaint (bad service, wrong order, quality inconsistency) as opposed to incidental "아쉬웠지만..." asides inside an otherwise glowing review (sold out item, small parking lot — these don't count). If a real complaint turns up despite a high average, don't grant the star — fold the complaint into the note's 注意事項 instead (see the 明浩豚肉 機場店 / 濟州따이 precedent).

This is per-place manual judgment on step 3, not a script — when adding one or two new places it's fast enough to just read the reviews inline; only build a batch/background script (§2 batch pattern) when checking many places at once.

## 6. Validation before committing

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

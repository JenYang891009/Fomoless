# iNCU 活動報名爬蟲

用自己瀏覽器已登入的 session，從「活動列表」＋「活動詳細資料」抓活動，
欄位依「FomoLess 活動 API 欄位需求（草案）」一、二節的表格（攤平成一層）。

## 準備 cookie（只放自己電腦，不要 commit、不要分享）
瀏覽器登入 → 活動報名頁 → F12 → Network → 點 `activity` 那筆 → Request Headers 的 Cookie 值，
存成同資料夾的 `ncu_cookies.txt`（一行）。

## 跑
```
py -m pip install requests beautifulsoup4
py portal_activity_crawler.py crawl              # 所有還沒結束的活動 → output/portal_activities_日期.json
py portal_activity_crawler.py crawl -n 5         # 測試：只抓 5 則
py portal_activity_crawler.py samples output/portal_activities_日期.json   # 挑 3 筆範例 → samples/
```

在程式裡用（之後包 Lambda）：
```python
from portal_activity_crawler import crawl
activities = crawl()   # list[dict]，所有還沒結束的活動
```

## 輸出欄位
| 欄位 | 來源 |
|---|---|
| activity_id, title, organizer, audience, url | 內頁上方 dl |
| contact_email | 固定 null（不存承辦人姓名、個人信箱；raw_fields 也不放） |
| raw_fields | 內頁畫面上顯示的所有「標籤→文字」原樣保留，活動內容在 `raw_fields["活動內容"]` |
| scraped_at | 這筆資料爬下來的時間（UTC） |
| image_url（list） | 列表卡片＋輪播＋內頁內容裡的圖 |
| status | 內頁場次按鈕（已截止…），沒有就用列表卡片標籤 |
| views | 列表卡片「瀏覽次數」 |
| sdgs_tag | 列表卡片「SDGs標籤」 |
| speaker, location, start_at/end_at, registration_start/end, capacity, waitlist_capacity, hour_tags, learning_passport_hours/condition, soft_skill_hours | 內頁場次區塊 |
| registered_count, waitlist_count | 內頁「報名人數：0 / 80」「備取人數：0 / 0」 |
| registration_method/url | 內容裡有 Google 表單等連結 → external，否則 portal |
| registration_status | 由 status 轉成：開放報名／已截止／已額滿／尚未開始 |
| updated_at, location_code | 網頁上沒有，固定 null |

## 每天執行
- 預設輸出所有還沒結束的活動（end_at 還沒過）；加 `--new-only` 只抓 `seen_ids.json` 裡沒有的
- Windows 工作排程器：
  `schtasks /create /tn FomoLessPortal /sc daily /st 08:00 /tr "cmd /c cd /d <資料夾> && py portal_activity_crawler.py crawl >> crawl.log 2>&1"`
- AWS：EventBridge 每天觸發 `lambda_handler`（回傳 crawl() 的結果）
- cookie 會過期，過期時程式會停下來提示重新複製

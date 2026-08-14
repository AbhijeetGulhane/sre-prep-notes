# Twitter — Write QPS + Storage Estimate (Cold Drill)

*Standalone capacity-estimation exercise, not attached to a full NALSD design doc (no Twitter/social-feed topic exists yet in `system-design/`). Given: 300M MAU, 50% daily active, 2 tweets/day average, storage = 280-char text + metadata.*

---

## 1. Daily active tweeting users
```
300M MAU × 50% daily active = 150M DAU
```

## 2. Daily tweet volume
```
150M DAU × 2 tweets/day = 300,000,000 tweets/day
```

## 3. Write QPS
```
Average: 300,000,000 ÷ 86,400 sec ≈ 3,472 tweets/sec
Peak (3x burst factor, consistent with this repo's convention): ≈ 10,400 tweets/sec
```

## 4. Storage per tweet
```
Text: 280 chars ≈ 280 bytes (approximating ~1 byte/char — generous for mostly-ASCII
      content; real UTF-8 could run higher for heavy emoji/CJK use — an assumption,
      not a precise figure)

Metadata: tweet_id (8B) + user_id (8B) + timestamp (8B) + reply/retweet/quote
          pointers (3×8B=24B) + engagement counters (3×4B=12B) + flags/misc (16B)
          ≈ 76 bytes, rounded to ~100 bytes for indexing overhead

Total: 280 + 100 = 380 bytes/tweet → rounded to 400 bytes/tweet for clean math
```

## 5. Annual storage, raw
```
300,000,000 tweets/day × 400 bytes × 365 days
= 43,800,000,000,000 bytes ≈ 43.8 TB/year (raw)
```

## 6. With 3x replication (standard durability assumption used throughout this repo)
```
43.8 TB × 3 ≈ 131.4 TB/year
```

---

## 7. Sanity check against a prior estimate

Total tweets/year here: **300M × 365 ≈ 109.5 billion** — remarkably close to [01.1-url-shortener-nalsd.md](../system-design/01.1-url-shortener-nalsd.md)'s **100 billion** record target, and at a similar per-record size (400 bytes here vs. 500 bytes there). Two independently-derived estimates landing in the same order of magnitude, for genuinely different systems, is a good cross-check that the methodology is sound rather than a fluke — worth doing this kind of cross-reference against a prior estimate whenever one's available, as its own interview habit.

## 8. The actual takeaway

**131 TB/year is tiny** — three to four orders of magnitude smaller than [09-video-platform.md](../system-design/09-video-platform.md), which landed in the exabyte range for a single year. Twitter *feels* like a bigger, more consequential platform than a URL shortener, but storage footprint tracks **payload type**, not brand prominence or user count — text is extraordinarily cheap compared to video, and that gap dwarfs any difference in company scale or fame. Worth having this contrast ready if an interviewer asks "why is this so much smaller than YouTube" — it's not that Twitter is a smaller system, it's that text and video are different orders of magnitude of data per unit of user activity.

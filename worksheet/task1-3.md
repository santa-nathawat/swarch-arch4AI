# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ:** Santa

**Student ID / รหัสนิสิต:** 6732010421

This is **individual work**: each student completes and submits their own copy, in their own fork.

*งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง*

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|
| t0-large | 492 | 586 | 93.5% | 100% |
| t0-small | 12 | 30 | 72.5% | 0% |

**Q0.** What does the large model buy, and what does it cost?

*large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร*

> The large model buys accuracy: 93.5% vs 72.5% for small (t0-large vs t0-small), about 21 points more. It costs latency (p50 492 vs 12 ms, p95 586 vs 30 ms) and money, since 100% of requests are large calls, and it can only do 20 calls/s.

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
|---|---|---|---|---|---|---|
| t1-before | not recorded | 240 | 419 | 21 | 0 | 92.7% |
| t1-after | not recorded | 387 | 1006 | 0 | 28 | 93.0% |

**Q1.1** In `t1-before`, p50 can look *better* than the baseline. Why is p50 misleading here? Which column tells the real story?

*ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง*

> In t1-before the 21 requests that hung got cut off as errors, so they are not in the fast part of the latency list, and p50 (240 ms) looks better than baseline (492). p50 only shows the middle request. The errors column (21 of 200 failed) tells the real story: about 10% of reviews just failed.

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

*การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway*

> A hung call doesn't fail fast, it holds a connection/worker in the review service and the gateway for up to 20 s. Those stuck requests queue up in front of the large model (only 4 at once), so healthy requests behind them also wait or time out. So one slow call slows down lots of other reviews, not just itself.

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

*รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น*

> Save them as pending (FR4) and have a background batch job re-score them with the large model later, e.g. from a queue, then update the label. That's the asynchronous / batch topology (T4).

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

*เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที*

> 0.1 s is shorter than a normal large call (~200 ms, p50 492 under load), so almost every review would go pending even when nothing is wrong. 10 s means a hung call still holds resources for 10 s and breaks QR2 (wait max 1 s), plus the client gives up at 10 s anyway so we'd get errors again.

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run | threshold | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|---|
| t2-0.6 | 0.6 | 6 | 541 | 91.5% | 38% |
| t2-0.7 | 0.7 | 371 | 583 | 93.0% | 53% |
| t2-0.8 | 0.8 | 503 | 711 | 93.5% | 64% |
| t2-0.9 | 0.9 | 479 | 644 | 93.5% | 77% |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

*จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป*

> I'd pick 0.7: accuracy 93.0% is basically the same as large (93.5%) and above QR3's 90%, but only 53% large calls instead of 100% (t2-0.7). 0.6 is cheaper (38%) but 91.5% is too close to the 90% limit. I give up a bit of accuracy (0.5 points) and a bit of tail latency vs always-small.

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

*ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก*

> p50 depends on what the middle request does. With 0.7, 53% still go to large so the median request is still a large call (p50 371 ms), while at 0.6 more than half stay on small and p50 drops to 6 ms. p95 is always a request that went to large, and it waited for small first, so p95 (583) is about the same as large-only (586). Cascade only speeds up the easy reviews.

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

*cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้*

> Sarcasm or negation like "oh great, cold food again" or "not bad at all": the small model can be confidently wrong, so the cascade never asks the large model. The user (owner) sees a wrong label, e.g. a complaint counted as positive, and the stats can't be trusted.

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls | cache hits |
|---|---|---|---|---|---|
| t3-cache | 6 | 564 | 93.0% | 38% | 35% |

**Q3.1** Which reviews benefit from the cache, and which never will?

*รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์*

> Reviews whose text repeats exactly (after lowercase/trim), like short ones "good", "too salty", or copy-pasted/spammed reviews, hit the cache (35% hits in t3-cache, large calls dropped 53% to 38%). Long unique reviews with their own details will basically never hit.

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

*สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร*

> The cache holds answers from the old model, so it must be cleared or invalidated, otherwise old labels keep being served. I'd put the model version in the cache key (e.g. model_v2 + normalized text) so a new deploy automatically misses old entries, and maybe add a TTL.

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

*ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture*

> That the average/p50 can lie: in t1-before p50 actually got better while 21 requests failed, and with the cascade p50 went down to 6 ms but p95 almost didn't move. You have to look at p95, errors and large-call % together, not just one number. Also small changes in threshold (0.6 to 0.7) change cost a lot (38% vs 53%).

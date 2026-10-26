> Agent: Claude Code · Model: Claude Opus 5.5 (`claude-opus-5-5`) · Task 3: ATAM review of `docs/adr/0002-publish-event-to-rabbitmq.md`

# Review: ADR-0002 Publish "order placed" event to RabbitMQ

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| S1 (Performance) | ลูกค้า | ส่ง `PlaceOrder` 200 orders/s | ช่วง 11.11 peak, MailFast ตอบช้า p99 8 s | checkout ตอบลูกค้าโดยไม่รอ MailFast | p95 ของ `PlaceOrder` ≤ 500 ms (การ publish ต้องใช้ไม่เกิน budget ประมาณ 200 ms ที่เหลือ) |
| S2 (Availability / correctness) | infrastructure | RabbitMQ broker หรือ `loyaltyservice` ล่ม | peak 200 orders/s | คำสั่งซื้อยังสำเร็จ; แต้มถูกมอบเมื่อระบบกลับมา | คำสั่งซื้อสำเร็จ 100 %; แต้มหาย 0, แต้มซ้ำ 0; มอบแต้มภายใน 1 นาทีหลัง order |
| S3 (Reliability) | MailFast | ล้มเหลว 0.5 % และจำกัด 50 emails/s | peak 200 orders/s, `emailservice` หลาย replica | buffer, retry, DLQ | อีเมลหาย 0; อัตราเรียก MailFast รวมทุก replica ≤ 50/s |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Risk | ถ้า publish ล้มเหลวหลังตัดบัตร ADR ให้ "retry ต่อ" ซึ่งเป็น retry ในหน่วยความจำ (Decision, บรรทัด 68) | เป็นจุดอ่อนเดียวกับที่ใช้ปฏิเสธ Option 2 ("pod restart แล้ว event หาย") ถ้า event หาย ลูกค้าจะไม่ได้แต้ม จึงทำ exactly once ไม่ได้ (S2) |
| Risk | ไม่ได้ระบุ timeout ของการ publish และ publisher confirm เมื่อ broker ล่มหรือช้า | ถ้า connection ค้าง `PlaceOrder` จะ block แบบเดียวกับปัญหา `sendOrderConfirmation` ที่ ADR ยกมาเอง คำกล่าวที่ว่า "checkout ยังสำเร็จแม้ broker ล่ม" จึงยังไม่มีอะไรรองรับ (S1, S2) |
| Risk | rate limit 50/s ทำเป็น "prefetch + rate limiter" ใน consumer แต่ละตัว | ถ้ามี N replica อัตรารวมจะเป็น N×50/s ส่วน prefetch คุมแค่จำนวนข้อความที่ค้างอยู่ ไม่ได้คุมอัตรา และการ retry ก็นับรวมใน limit ด้วย (S3) |
| Sensitivity point | ระยะเวลาของ peak ใช้สมมติฐาน 1 นาที (บรรทัด 31, 109) | ขนาด queue และเวลาระบายอีเมล (9,000/50 ประมาณ 3 นาที) เพิ่มตามระยะเวลาของ peak แบบเชิงเส้น ถ้า peak ยาว 30 นาที จะมีอีเมลค้างประมาณ 270,000 ฉบับ และต้องใช้เวลาระบายประมาณ 90 นาที |
| Sensitivity point | latency ของ publish + confirm และจำนวน consumer ของ `loyaltyservice` | ค่าแรกกำหนดว่า S1 ผ่านหรือไม่ ค่าหลังกำหนดว่าแต้มจะถึงภายใน 1 นาทีหรือไม่ แต่ทั้งสองค่ายังไม่มีการวัด (Open questions บรรทัด 134) |
| Trade-off point | "order สำเร็จแม้ publish ล้มเหลว" (availability) กับ "แต้ม exactly once" (correctness) | ถ้าไม่มี outbox สองเป้าหมายนี้ขัดกันโดยตรง ADR ยอมเสีย correctness ไปโดยไม่ได้เขียนไว้ชัด |
| Trade-off point | quorum queues 3 node + publisher confirms | ได้ durability เพิ่ม แต่เสีย latency บน critical path และเพิ่มภาระ operability ให้ทีม 4 คน |
| Non-risk | แยก durable queue ต่อ consumer ผ่าน exchange `orders` | MailFast ช้าหรือล่มจะไม่กระทบ loyalty หรือ analytics และเพิ่ม SMS หรือ fraud scoring ได้โดยไม่ต้องแก้ `checkoutservice` |
| Non-risk | เลือก RabbitMQ แทน Kafka | ปริมาณ 200 msg/s เล็กมาก และทีมมี 2 คนที่เคยใช้ RabbitMQ จึงสอดคล้องกับ driver 5 |
| Non-risk | ใช้ `order_id` (UUID) เป็น idempotency key ใน `loyaltyservice` | เป็นทางที่ถูกต้องสำหรับ at-least-once แต่มีเงื่อนไขว่าการบันทึก key กับการมอบแต้มต้องอยู่ใน transaction เดียวกัน (ADR ยังไม่ได้เขียนไว้) |

## 3. Missing or weak

- **Options ที่ไม่ได้พิจารณา:** (a) transactional outbox หรือ CDC เป็นทางเลือกหลัก ไม่ใช่แค่ open question (b) Redis Streams บน Redis ที่ทีมดูแลอยู่แล้วตาม ADR-0001 ซึ่งไม่ต้องเพิ่ม component stateful ใหม่ (c) hybrid: ส่ง loyalty ผ่าน queue แต่ analytics ใช้ fire-and-forget
- **Consequences ที่ไม่ได้ระบุ:** อีเมลซ้ำเมื่อ retry (มีอยู่แค่ใน Open questions); ต้องมี schema/versioning ของ `OrderPlaced` เมื่อ consumer เพิ่มขึ้น; ใครเป็นผู้ดูแล DLQ และ replay
- **Claims ที่อ่อนหรือน่าจะผิด:**
  - "เป็นตัวเลือกเดียวที่ตอบ driver 1–3" (บรรทัด 75): ADR ยอมรับเองว่า Kafka ก็ตอบได้ และ Option 3 ที่ไม่มี outbox ก็ยังตอบ driver 3 ได้ไม่ครบ
  - "Kafka ต้องเขียน rate limit เองมากกว่า": RabbitMQ เองก็ไม่มี rate limit ต่อ consumer ในตัว (ดู Risk ข้อ 3)
  - Validation "p100 < 1 นาที" ขัดกับ chaos test ที่ปิด RabbitMQ เพราะถ้า broker ล่มนานเกิน 1 นาทีก็ทำ SLA ไม่ได้ ADR ควรเขียนว่ายอมรับกรณีนี้หรือไม่
  - มีชื่อผู้ตัดสินใจใน Deciders แต่ Open questions ยังถามว่า "ใครเป็นผู้ตัดสินใจ" จึงขัดกันเอง
  - Diagram ไม่มี `frontend` (`Rel(customer, checkout, ..., "via frontend")`) และไม่มีการล้าง `cartservice` ที่อยู่ใน flow เดิม

## 4. Verdict: **Accept with changes**

1. **ตัดสินใจเรื่อง outbox ให้ชัด:** จะเพิ่ม store ให้ `checkoutservice` เพื่อทำ transactional outbox หรือจะยอมรับอย่างเป็นทางการว่าแต้มอาจหายได้ พร้อม reconciliation job ระหว่าง order กับแต้ม เพราะ retry ในหน่วยความจำมีจุดอ่อนแบบเดียวกับ Option 2 ที่ถูกปฏิเสธไป
2. **กำหนด publish timeout** (เช่น ≤ 50 ms) และพฤติกรรมเมื่อ broker ล่ม แล้ววัดว่า publish + confirm อยู่ใน budget 200 ms ภายใต้ load test 200 orders/s
3. **ทำ rate limiter แบบ global ข้าม `emailservice` replica** (หรือกำหนด replica = 1 และนับ retry รวมใน limit) แล้วระบุว่า idempotency ของ `loyaltyservice` ต้องอยู่ใน transaction เดียวกับการมอบแต้ม

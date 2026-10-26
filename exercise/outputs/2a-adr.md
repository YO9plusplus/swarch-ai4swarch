> Agent: Claude Code — Model: Claude Opus 5.5 (`claude-opus-5-5`) — Run: a (zero-shot, `context: none`)

# ADR-0002: ใช้ event "OrderPlaced" ผ่าน message broker เพื่อกระจายงานหลังการสั่งซื้อ

- **Status:** Proposed
- **Date:** 2026-10-21
- **Deciders:** Not evidenced in the provided files (ดู Open questions)

## Context and problem

Requirement R-12 กำหนดว่า หลังจากลูกค้าสั่งซื้อใน Online Boutique ระบบต้องทำ 3 อย่าง:

1. ส่งอีเมลยืนยันคำสั่งซื้อผ่าน **MailFast** (ผู้ให้บริการอีเมลภายนอกรายใหม่) แทน logic จำลองที่อยู่ใน `emailservice`
2. มอบแต้มสะสม (loyalty points) ผ่าน `loyaltyservice` ซึ่งยังต้องสร้างใหม่
3. บันทึก analytics event "มีการสั่งซื้อ" ให้ทีมการตลาด

ปัญหาที่ต้องตัดสินใจ: **บริการที่รับคำสั่งซื้อควรแจ้ง consumer ทั้ง 3 รายอย่างไร** ให้การสั่งซื้อไม่ช้าลงหรือล้มเหลวเพราะงานหลังการสั่งซื้อ และเพิ่ม consumer ใหม่ได้ง่ายในอนาคต

ข้อเท็จจริงที่ทราบ:
- มี `emailservice` อยู่แล้ว และ `loyaltyservice` ยังไม่มี (จาก requirement)
- MailFast เป็นระบบภายนอก จึงควบคุม availability และ latency ของมันไม่ได้
- **[Inferred]** มีบริการหนึ่งที่จัดการขั้นตอนการสั่งซื้อและเรียก `emailservice` อยู่ในปัจจุบัน แต่ไม่มีไฟล์ให้ยืนยันชื่อ protocol หรือวิธีเรียก (Not evidenced in the provided files)
- ยังไม่มีตัวเลขเรื่องปริมาณคำสั่งซื้อ latency budget หรือ SLA ของ MailFast (ดู Open questions)

## Decision drivers

1. **Availability ของการสั่งซื้อ:** การสั่งซื้อต้องสำเร็จได้ แม้ MailFast, `loyaltyservice` หรือระบบ analytics จะล่มอยู่
2. **Performance (latency ของ checkout):** งานหลังการสั่งซื้อไม่ควรเพิ่มเวลาตอบกลับลูกค้า
3. **Modifiability:** เพิ่มหรือเปลี่ยน consumer (เช่น เปลี่ยนผู้ให้บริการอีเมล หรือ consumer ใหม่ของทีมการตลาด) ได้โดยไม่ต้องแก้บริการที่รับคำสั่งซื้อ
4. **Reliability ของแต่ละงาน:** อีเมลและแต้มสะสมต้องไม่หาย และไม่ถูกทำซ้ำ (เช่น ไม่ให้แต้มสองครั้ง)
5. **Cost และ operational complexity:** ไม่ควรเพิ่ม infrastructure เกินกว่าที่ทีมดูแลไหว

## Considered options

1. **A: เรียกแบบ synchronous** บริการที่รับคำสั่งซื้อเรียก `emailservice` (ซึ่งเรียก MailFast), `loyaltyservice` และ analytics ทีละตัวโดยตรง (เช่น gRPC/HTTP) ระหว่างการสั่งซื้อ
2. **B: Publish/subscribe ผ่าน message broker** บริการที่รับคำสั่งซื้อ publish event `OrderPlaced` เพียงครั้งเดียวไปยัง broker (เช่น RabbitMQ, Kafka หรือ managed pub/sub) แล้ว `emailservice`, `loyaltyservice` และ analytics consumer แต่ละตัว subscribe แยกกัน
3. **C: ยิงแบบ asynchronous ภายในบริการเดียว (fire-and-forget)** ใช้ background thread หรือ goroutine ในบริการที่รับคำสั่งซื้อเรียกทั้ง 3 ปลายทางหลังตอบลูกค้าแล้ว โดยไม่มี broker

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A: Synchronous calls | ง่ายที่สุด ไม่ต้องเพิ่ม infrastructure; รู้ผลทันทีและ debug ง่าย | checkout ช้าลงเท่ากับผลรวม latency ของทุกปลายทาง (รวม MailFast); ปลายทางใดล่มก็ทำให้การสั่งซื้อล้มเหลวหรือต้องเขียน error handling ซับซ้อน; coupling สูง เพราะเพิ่ม consumer ทีไรต้องแก้บริการที่รับคำสั่งซื้อทุกครั้ง |
| B: Pub/sub via broker | checkout ไม่ต้องรอ consumer; consumer ล่มได้โดยไม่กระทบการสั่งซื้อ และ event ค้างรอใน queue; เพิ่ม consumer ใหม่ได้โดยไม่แก้ producer; retry และ dead-letter ทำแยกต่อ consumer ได้ | ต้องเพิ่มและดูแล broker (cost, ops, monitoring); เป็น eventual consistency คือแต้มและอีเมลมาช้ากว่าการสั่งซื้อ; ต้องออกแบบ idempotency และ event schema; ยังมีปัญหา dual-write ระหว่าง "บันทึก order" กับ "publish event" ถ้าไม่ใช้ outbox |
| C: In-process async | ไม่ต้องเพิ่ม infrastructure; ไม่เพิ่ม latency ให้ลูกค้า | ถ้า pod restart ระหว่างทาง event จะหาย (ไม่มี durability); retry ต้องเขียนเองในบริการเดียว; coupling ยังสูงเหมือน A; บริการที่รับคำสั่งซื้อต้องรู้จักทุก consumer |

## Decision

เลือก **Option B: publish event `OrderPlaced` ผ่าน message broker** โดยมี consumer 3 ตัวดังนี้:

- `emailservice`: เปลี่ยน logic จำลองเป็นการเรียก MailFast API มี retry แบบ backoff และ dead-letter queue
- `loyaltyservice` (ใหม่): คำนวณและบันทึกแต้มสะสม โดยใช้ order ID เป็น idempotency key
- analytics consumer: ส่งต่อ event ให้ระบบของทีมการตลาด

เหตุผล: Option B ตอบ driver 1–3 (availability, performance, modifiability) ได้ดีที่สุด เพราะการสั่งซื้อไม่ขึ้นกับ MailFast หรือบริการใหม่ที่ยังไม่เสถียร และเพิ่ม consumer ได้โดยไม่แก้ producer ส่วน driver 4 แก้ด้วย durable queue ร่วมกับ idempotent consumer แลกกับ cost/complexity (driver 5) ที่สูงขึ้น ส่วนการเลือก broker ตัวจริงยังเปิดไว้ (ดู Open questions)

## Consequences

- **Positive:** latency ของ checkout ไม่เพิ่มตามจำนวน consumer หรือความช้าของ MailFast (quality attribute: performance)
- **Positive:** การสั่งซื้อยังสำเร็จได้แม้ MailFast, `loyaltyservice` หรือ analytics ล่ม เพราะ event รอใน queue (quality attribute: availability)
- **Positive:** เพิ่ม consumer ใหม่ หรือเปลี่ยนผู้ให้บริการอีเมลในอนาคต โดยไม่แก้บริการที่รับคำสั่งซื้อ (quality attribute: modifiability)
- **Negative:** ต้องติดตั้ง ดูแล และ monitor broker เพิ่ม (quality attribute: cost, operability)
- **Negative:** อีเมลและแต้มสะสมเป็น eventual consistency ลูกค้าอาจเห็นแต้มช้ากว่าการสั่งซื้อ (quality attribute: usability/consistency)
- **Negative:** tracing ข้ามบริการแบบ async ยากขึ้น ต้องส่ง correlation ID ไปใน event (quality attribute: testability/observability)
- **Risks and mitigations:**
  - *Event หาย หาก order บันทึกแล้วแต่ publish ไม่สำเร็จ (dual-write)* → ใช้ transactional outbox หรือ publish พร้อม retry (ขึ้นกับว่ามี order database หรือไม่ ซึ่ง Not evidenced in the provided files)
  - *ส่งอีเมลซ้ำหรือให้แต้มซ้ำเพราะ at-least-once delivery* → consumer ต้อง idempotent โดยใช้ order ID เป็น key
  - *MailFast ล่มหรือติด rate limit* → retry แบบ exponential backoff, dead-letter queue และ alert
  - *Event schema เปลี่ยนแล้วทำให้ consumer พัง* → กำหนดเวอร์ชันของ schema `OrderPlaced` และเพิ่ม field แบบ backward-compatible เท่านั้น
  - *ข้อมูลส่วนบุคคล (อีเมลลูกค้า) อยู่ใน event ที่ทีมการตลาดใช้* → analytics consumer รับเฉพาะ field ที่จำเป็น หรือ mask PII

## Validation

- **Load test:** วัด p95 latency ของ checkout ก่อนและหลังการเปลี่ยนแปลง (ยังไม่มีเป้าหมายตัวเลข ดู Open questions)
- **Fault injection:** ปิด MailFast (ใช้ mock) และ `loyaltyservice` ระหว่างทดสอบ แล้วยืนยันว่าการสั่งซื้อยังสำเร็จ และเมื่อบริการกลับมา อีเมลและแต้มถูกประมวลผลครบโดยไม่ซ้ำ
- **Metrics:** queue depth, consumer lag, จำนวนข้อความใน dead-letter queue, อัตราส่งอีเมลสำเร็จ และเวลาตั้งแต่สั่งซื้อจนอีเมลถึง/แต้มเข้า
- **Review date:** ทบทวน ADR นี้หลังใช้งานจริงตามระยะที่ทีมกำหนด (ยังไม่ระบุ)

## Open questions

- ปริมาณคำสั่งซื้อ (ปกติและช่วง peak) และ latency budget ของ checkout คือเท่าใด
- MailFast มี SLA, rate limit, รูปแบบ API (REST/SMTP/webhook) และค่าใช้จ่ายต่อข้อความอย่างไร
- อีเมลยืนยันต้องถึงลูกค้าภายในเวลาเท่าใด และแต้มสะสมต้องแสดงทันทีหรือรอได้
- ปัจจุบันบริการใดเรียก `emailservice` ด้วย protocol อะไร และมี order database หรือไม่ (มีผลต่อการทำ outbox)
- มี message broker ในระบบหรือ infrastructure อยู่แล้วหรือไม่ และทีมมีทักษะกับ broker ตัวใด (RabbitMQ, Kafka, managed pub/sub)
- `loyaltyservice` ต้องเก็บข้อมูลแต้มที่ใด และกฎการคำนวณแต้มคืออะไร
- ทีมการตลาดใช้ระบบ analytics ใด รับ event รูปแบบใด และต้องการข้อมูล field ใดบ้าง (รวมข้อกำหนดเรื่อง PII/PDPA)
- ใครเป็น Deciders ของ ADR นี้

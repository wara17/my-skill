# Demo Script: codebase-summary-kotlin-springboot + dev-workflow

ตัวอย่างใช้ domain **Book** (จัดการหนังสือ) — ถ้าคุณใช้ domain อื่น แทนชื่อ resource/field ได้เลย โครง requirement เหมือนเดิม

---

## Precondition (ต้องมีก่อน demo)

Repo Spring Boot + Kotlin ที่มี:
- `Book` entity: `id`, `title` (String, not null), `isbn` (String, not null, unique), `price` (BigDecimal, not null)
- `BookController` มี endpoint พื้นฐานอย่างน้อย `GET /api/books`, `POST /api/books`
- `@ControllerAdvice` + response structure กลาง เช่น `ErrorResponse(code, message, fieldErrors)`
- (แนะนำ) ใส่หัวข้อ **"⚠️ Watch out"** ใน `docs/codebase/CODEBASE.md` หลัง generate เสร็จ ระบุ constraint จริงไว้ล่วงหน้า เช่น "Book.isbn เป็น NOT NULL ไม่มี default ห้ามทำให้ optional โดยไม่ migrate DB" — ใช้สำหรับ demo case 4

---

## Case 1 — สร้าง Codebase

พิมพ์:
```
สร้าง codebase summary ให้หน่อย
```
คาดว่า: trigger `codebase-summary-kotlin-springboot` → Generate mode (ครั้งแรกไม่มีไฟล์) → ได้ `docs/codebase/CODEBASE.md` + ผูก pointer ใน `AGENTS.md`

---

## Case 2 — เพิ่ม API ทั่วไป (ไม่มี blocker, ทดสอบ fast path)

พิมพ์:
```
เพิ่ม endpoint GET /api/books/{id} คืนข้อมูลหนังสือเล่มเดียวตาม id
ถ้าไม่เจอ id ให้ return 404 พร้อม error structure ที่มีอยู่แล้วในระบบ
```
คาดว่า: ไม่มี blocker, self-contained (ไฟล์เดียว/module เดียว) → เข้า **fast path** ทันที ไม่หยุดถาม
โชว์ทีม: เปิด `.task/{slug}-ac.md` ที่ถูกสร้างขึ้นแม้งานเล็กแค่นี้ + Assumptions list ที่โผล่มาก่อนโค้ด (แม้จะว่างก็ต้องมี list "none")

---

## Case 3 — แก้ API แบบ spec ครบ (ทดสอบ plan confirmation / AC ละเอียด)

พิมพ์:
```
แก้ POST /api/books ให้ validate request ก่อนบันทึก:
- title ห้ามว่าง หรือมีความยาวเกิน 255 ตัวอักษร
- isbn ต้องเป็นตัวเลข 13 หลัก และห้ามซ้ำกับที่มีอยู่แล้วในระบบ
- price ต้องมากกว่า 0

ถ้า validate ไม่ผ่าน ให้ return HTTP 400 พร้อม ErrorResponse ที่มีอยู่แล้ว
(code, message, fieldErrors) โดย fieldErrors ต้องระบุชื่อ field ที่ผิดแต่ละตัว
```
คาดว่า: AC ชัด ไม่มี blocker แต่ถ้าแตะหลายไฟล์ (controller + service + validator) อาจไม่เข้าเกณฑ์ "self-contained" → หยุดที่ Step 2+3 รอ **confirm plan** ก่อน implement
โชว์ทีม: เทียบกับ case 2 — งานใหญ่ขึ้นนิดเดียวก็เปลี่ยนพฤติกรรมเป็นหยุดถามแทน

---

## Case 4 — แก้ API แบบ spec ขาดข้อมูล (ทดสอบ blocker rule)

พิมพ์:
```
เพิ่ม field discountCode ใน POST /api/books เป็น optional string
ไม่บังคับกรอกก็ได้ ถ้าไม่ส่งมาก็ไม่ต้องเก็บอะไรพิเศษ
```
**เจตนา**: ให้ขัดกับ DB constraint จริง (ถ้าตั้ง `discountCode` เป็น column NOT NULL ไม่มี default ใน entity/migration) หรือขัดกับ "Watch out" ใน CODEBASE.md ที่เตรียมไว้
คาดว่า: เข้าเกณฑ์ blocker ("spec ขัดกับ DB/type constraint") → **หยุดถามก่อน** ไม่เดาเอง ไม่ implement ทันที

ตัวอย่างที่ **ไม่ใช่** blocker (เผื่ออยากโชว์เทียบ): "ลืมบอกว่า error message ตอน validate fail ควรเขียนว่าอะไร" → อันนี้แค่ edge case เล็ก จะไม่ถาม จะ infer เองแล้วโชว์เป็น assumption แทน

---

## Case 5 — เช็ค AC ทุกเคส

หลังแต่ละเคส เปิดไฟล์:
```
.task/{slug}-ac.md
```
ดู checkbox:
- `- [x] ... — auto-verified` = มี test คลุมและผ่าน
- `- [ ] ... — needs manual verification` = ยังไม่มี test คลุม ต้องเช็คเอง

ใช้เทียบทั้ง 3 เคส (2, 3, 4) ให้เห็นว่าทุกเคสมีไฟล์นี้เกิดขึ้นจริง ไม่ใช่แค่ agent พูดว่า "เสร็จแล้ว"

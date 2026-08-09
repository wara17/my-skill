# AC-Driven Agent Harness: codebase-summary-kotlin-springboot + dev-workflow

> เอกสารนี้เตรียมไว้สำหรับทำสไลด์ share ทีม — แต่ละหัวข้อ (##) ออกแบบให้แม็ปเป็น 1 สไลด์ได้เลย

---

## พื้นฐานก่อนเข้าเรื่อง

AI model แต่ละเจ้า (Claude, GPT ฯลฯ) มี **harness** ของตัวเองอยู่แล้ว — คือระบบเบื้องหลังที่ผู้ผลิตสร้างไว้ ทำให้ model ธรรมดากลายเป็น agent ที่ลงมือทำงานได้จริง (เรียก tool, อ่าน/เขียนไฟล์, รัน command) พูดง่ายๆ harness คือ **เครื่องยนต์** ที่ติดตั้งมาให้แล้ว ทีมเราไม่ได้สร้างเครื่องยนต์ใหม่

สิ่งที่ทีมเราทำคือใส่ **skill** เข้าไปกำหนดทิศทางให้เครื่องยนต์นั้นวิ่งไปแบบที่ทีมเราต้องการ — เปรียบเหมือน skill เป็น **GPS หรือพวงมาลัย** บอกเครื่องยนต์ว่าให้ไปทางไหน ไม่ได้เปลี่ยนเครื่องยนต์ แค่กำหนดเส้นทางให้ชัดขึ้น

**"Skill"** คือไฟล์ markdown ที่เขียนขั้นตอน/กติกาไว้ล่วงหน้า ระบบโหลดเข้า context ของ agent อัตโนมัติเมื่อ task ตรงเงื่อนไข ไม่ใช่โค้ดที่ compile แค่เป็น "คำสั่งมีโครงสร้าง" ให้ agent อ่านแล้วทำตาม

ต่างจาก autocomplete/chat ทั่วไปตรงที่เก็บ state ลงไฟล์จริง (`CODEBASE.md`, `.task/*.md`) — งานที่ context หลุดหรือข้าม session ยังมีความจำอยู่ ไม่เหมือน chat ที่ความจำหายเมื่อ conversation ยาวเกิน

---

## ทำไมถึงต้องมี 2 skill นี้

เวลาให้ agent ช่วยแก้โค้ดแบบปล่อยอิสระ (พิมพ์ requirement คุยไปเรื่อยๆ) จะเจอ 3 ปัญหาซ้ำๆ:

1. **Scan repo ใหม่ทุกครั้ง** — ทุกงานต้องมานั่งไล่โครงสร้าง entity, controller, convention ใหม่หมด ช้าและได้ความเข้าใจไม่เท่ากันในแต่ละรอบ
2. **ข้าม Acceptance Criteria** — งานเล็กๆ มักถูกมองว่า "ไม่ต้องคิดเยอะ" แล้วโค้ดตรงจากความเข้าใจ ไม่มีอะไรยืนยันว่าทำครบตามที่ขอจริงไหม
3. **หลุด context กลางทาง** — งานยาวๆ context ถูก compact ตัดข้อมูลออก agent ลืมว่าเคยตกลง requirement ไว้ว่าอะไร

แนวทางแก้คือแยกเป็น 2 skill ที่ทำงานร่วมกัน: อันหนึ่งสร้าง "ความจำของ repo" ไว้ล่วงหน้าครั้งเดียว อีกอันบังคับให้ทุกงาน implement เดินตาม pipeline เดียวกันเสมอ

**นี่ไม่ใช่ TDD** — ไม่ได้เขียน test ก่อนโค้ดเพื่อออกแบบ แต่เป็น **agent harness แบบ AC-driven**: scaffolding ที่ครอบ agent ไว้ให้เดินตามขั้นตอนที่ควบคุมได้ ใช้ไฟล์ (`CODEBASE.md`, `.task/*.md`) เป็นความจำถาวรแทนการพึ่ง conversation memory ที่หลุดได้เมื่อ context ยาว

---

## Skill 1: codebase-summary-kotlin-springboot

**ทำอะไร**: สแกน repo Kotlin/Spring Boot ครั้งเดียว สร้างไฟล์ `CODEBASE.md` (~150 บรรทัด) ให้ทุกงานถัดไปอ่านแทนการ scan ซ้ำ

**ขั้นตอน**:
1. Mode check — มีไฟล์อยู่แล้วหรือยัง (ถ้ามี → ไป Update mode ไม่ scan ใหม่ทั้งหมด)
2. สแกน 7 หัวข้อ: build/config, entry point/layering, API layer, service/domain, data layer, cross-cutting (security/integration/testing), conventions
3. สร้างไฟล์ตาม template คงที่ (Stack, Structure, API surface, Domain & Service, Data layer, Testing, Security & Integrations, Conventions, Watch out)
4. ผูก pointer ไว้ใน `AGENTS.md`/`CLAUDE.md` เพื่อให้ agent อื่นรู้ว่าต้องอ่านไฟล์นี้ก่อน

**Update mode**: ไม่ scan ใหม่ทั้ง repo — diff จาก commit sha เดิม แล้วแพตช์เฉพาะ section ที่เปลี่ยน ถ้าเปลี่ยนเกิน ~30% ของ repo ถึงจะแนะนำให้ generate ใหม่ทั้งหมด

---

## Skill 2: dev-workflow

**ทำอะไร**: pipeline มาตรฐาน 6 ขั้นสำหรับทุกงาน implement/fix ไม่ว่า requirement จะมาจาก Jira, PR description หรือพิมพ์เองในแชท

**ขั้นตอน**:
1. **Requirement + Gap Check + AC** — สรุป requirement, ดึง AC เป็น checklist, เทียบกับ CODEBASE.md, บันทึกลง `.task/{slug}-ac.md` เสมอ (แม้งานเล็กสุดก็ต้องมีไฟล์นี้)
2. **Explore + Plan** — อ่าน CODEBASE.md เฉพาะส่วนที่เกี่ยว เขียนแผนพร้อมอ้างอิงที่มา + ระบุว่าแต่ละ AC จะมี test อะไรมาคลุม
3. **Implement** — ทำตามแผน พร้อมเขียน/แก้ unit test คู่กันไปเลย ไม่ใช่ทำทีหลัง
4. **Test** — รัน lint ก่อน แล้วรัน test suite ตามคำสั่งใน CODEBASE.md, retry ได้สูงสุด 2 ครั้งถ้า fail
5. **เช็ค AC ทีละข้อ** — อัปเดต `.task/{slug}-ac.md` เป็น auto-verified (มี test คลุมและผ่าน) หรือ needs manual verification (ยังไม่มี test คลุม)
6. **Summary** — สรุปไฟล์ที่แก้ + ตาราง AC pass/fail สำหรับใส่ใน PR description

**Blocker rule**: หยุดถาม user ก่อนโค้ด เฉพาะกรณีเสี่ยงจริงๆ 3 แบบเท่านั้น — spec ขัดกับ DB/type constraint, spec ขัดกับ "Watch out" ใน CODEBASE.md, หรือ spec อ้างถึงส่วนที่ไม่มีอยู่จริง นอกนั้น infer เองแล้วโชว์เป็น assumption ให้เห็น ไม่ถามจุกจิก

---

## Skill 3: requirement-version-resolver (ใช้เมื่อ requirement มาจาก Jira/Confluence)

**ทำอะไร**: dev-workflow เรียก skill นี้เป็น Step 1-pre เฉพาะกรณี requirement source เป็น Jira ticket หรือ Confluence page — ถ้า user พิมพ์ requirement เองในแชท ข้าม skill นี้ไปเลย ไม่ต้องเรียก

**ปัญหาที่แก้**: การ์ด Jira/Confluence หลายทีมมีการ mark เวอร์ชันด้วยสี/ขีดฆ่า (เช่น เขียวทับ = เพิ่มมาทีหลัง, แดงขีดฆ่า = ถูกลบออกในเวอร์ชันถัดมา) ถ้า agent อ่าน raw text ตรงๆ โดยไม่ resolve ก่อน อาจงงว่าส่วนไหนคือ requirement ที่ใช้จริง ณ ตอนนี้ นำไปเป็น requirement ที่ implement ผิดได้

**ขั้นตอน**:
1. หา MCP tool ที่ใช้ดึง Jira/Confluence จริง (ไม่เดาชื่อ tool) เลือกตัวที่คืนข้อมูล raw/structured (ADF หรือ storage format) ไม่ใช่ตัวที่คืนแบบ rendered/plain text ที่สี/ขีดฆ่าถูกตัดออกไปแล้ว
2. ระบุ target version — ถามครั้งเดียวถ้า user ไม่ได้บอก
3. หา legend ว่าสีแต่ละสีหมายถึงอะไร — **ไม่เดาเอง** (เขียวไม่ได้แปลว่า "เพิ่ม" เสมอไปในทุกทีม) ถ้าไม่มี legend หรือความหมายไม่ชัด หยุดถาม user ก่อน
4. ไล่ resolve ทุกส่วนที่ mark ไว้ตาม target version → เก็บ/ตัดออกตามกฎ (เช่น ของที่ถูกลบไปแล้วก่อนเวอร์ชันเป้าหมาย → ตัดออก)
5. Output เป็น plain text ที่ resolve ครบแล้ว ไม่มีสี/ขีดฆ่าให้ agent step ถัดไปต้องตีความเอง

**Caching**: เก็บผล resolve ไว้ตาม source + target version เช็ค last-modified/version ของหน้าก่อน ถ้าไม่เปลี่ยนก็ใช้ผลเดิมซ้ำ ไม่ต้อง resolve ใหม่ทุกครั้งที่เปิดงาน

---

## ลดอะไรไปได้บ้าง

| ปัญหาเดิม | ลดด้วยอะไร |
|---|---|
| Scan repo ซ้ำทุกงาน (เสียเวลา + เข้าใจไม่เท่ากันแต่ละรอบ) | `CODEBASE.md` สร้างครั้งเดียว อัปเดตแบบ diff |
| งานเล็กมักไม่มี AC ชัด ทำเสร็จแบบ "ดูดีก็จบ" | AC checklist บังคับทุกงาน แม้ bug fix 1 บรรทัด |
| Context หลุดกลางงานยาว agent ลืม requirement | ไฟล์ `.task/*.md` เป็นความจำถาวรที่ไม่หลุดตาม context |
| Assumption ที่ agent เดาเอง ไม่มีใครเห็น | บังคับโชว์ "Assumptions list" ก่อนเริ่มโค้ดทุกครั้ง |
| ถามคำถามจุกจิกจนช้า หรือไม่ถามเลยจนพลาด | Blocker rule ชัดเจนว่าเมื่อไหร่ต้องถาม เมื่อไหร่ไม่ต้อง |
| Test ไม่ครอบคลุม AC ใหม่ | บังคับเขียน test คู่กับ implement ไม่ใช่แค่รัน test เดิม |
| ไม่รู้ว่า AC ไหนยัง verify ไม่ได้จริง | ไฟล์ AC ถูก mark auto-verified / needs manual verification แยกชัด |
| อ่าน requirement จาก Jira ผิด เพราะสี/ขีดฆ่าที่ track เวอร์ชันยังไม่ resolve | `requirement-version-resolver` แปลงเป็น plain text ที่ resolve ตาม target version แล้วก่อนส่งต่อ |

---

## ข้อดี

- **Consistency** — ทุกงานเดินตาม pipeline เดียวกัน ไม่ขึ้นกับว่าใครเป็นคน prompt หรือ mood วันนั้น
- **Traceability** — มีไฟล์ `.task/*.md` และ `CODEBASE.md` เป็นหลักฐานว่าทำอะไรไปบ้าง ไม่ใช่แค่ agent พูดว่า "เสร็จแล้ว"
- **ทนต่อ context loss** — งานยาวๆ context หลุดได้ แต่ state สำคัญอยู่ในไฟล์ ไม่ใช่ conversation memory
- **ถามเฉพาะตอนจำเป็นจริง** — ไม่ถามทุกอย่าง (ช้า) และไม่เดาทุกอย่าง (เสี่ยง) — มี threshold ชัด
- **Scale ได้กับ repo ใหญ่** — Update mode ใช้ diff แทนการ scan ใหม่ทั้งหมด, รองรับ split เป็นหลายไฟล์ถ้า repo ใหญ่มาก

## ข้อเสีย / ข้อจำกัด

- **Overhead กับงานเล็กมากๆ** — แม้แต่ typo fix ก็ต้องผ่าน AC extraction + สร้างไฟล์ `.task/*.md` อาจรู้สึกหนักเกินไปสำหรับงาน 1 บรรทัด
- **พึ่ง CODEBASE.md ที่ทันสมัย** — ถ้าไม่มีคน trigger update เป็นประจำ ไฟล์จะเก่ากว่าโค้ดจริง แล้ว agent จะ plan บนข้อมูลผิด
- **Blocker rule ยังต้องอาศัยการตีความ** — เส้นแบ่งระหว่าง "blocker" กับ "not a blocker" ไม่ได้ชัดเจน 100% ในทุกกรณี ขึ้นกับ agent ตีความสถานการณ์
- **ไม่ครอบคลุม git workflow** — dev-workflow เตรียม PR description ให้ แต่ไม่ commit/push/เปิด PR เอง ยังต้องมีคนทำส่วนนั้น
- **Auto-verified ขึ้นกับคุณภาพ test ที่เขียน** — ถ้า test ที่เขียนคู่ implementation ไม่ได้ตรวจ edge case จริง AC จะโดน mark auto-verified ทั้งที่ยังไม่ปลอดภัย 100% — ยังต้องมี manual review อยู่ดี
- **ไม่ใช่ตัวแทน code review ของคน** — ลด manual work บางส่วน แต่ไม่ได้แทนที่ senior review ทั้งหมด

---

## สรุป 1 บรรทัดต่อ skill (ไว้ใส่สไลด์ title)

- `codebase-summary-kotlin-springboot` = **ความจำของ repo ที่ทุก agent อ่านร่วมกันได้**
- `dev-workflow` = **pipeline บังคับให้ทุกงาน implement มี AC, มี plan, มี test, มีหลักฐานว่าทำครบ**
- `requirement-version-resolver` = **ตัวแปล Jira/Confluence card ที่มี markup สีให้เป็น requirement สะอาดๆ ก่อน implement**

---

## คำถามที่น่าจะโดนถาม (เตรียมคำตอบไว้ล่วงหน้า)

**Q: ถ้า CODEBASE.md เก่ากว่าโค้ดจริงล่ะ agent จะรู้ได้ยังไง?**
A: ไม่รู้เองอัตโนมัติ 100% — พึ่งการ trigger update เป็นระยะ (หลัง PR ใหญ่ merge หรือ weekly) ถ้าไม่มีใคร trigger ไฟล์จะเก่ากว่าโค้ดจริงและ agent จะ plan บนข้อมูลผิด นี่คือข้อจำกัดที่ต้องมี owner/routine ดูแล ไม่ใช่ set-and-forget

**Q: AI เขียน test เอง จะมั่นใจได้ยังไงว่า test ดีจริง ไม่ใช่แค่เขียนให้ผ่าน?**
A: ไม่ได้มั่นใจ 100% — "auto-verified" แปลว่ามี test คลุมและผ่าน ไม่ได้แปลว่า test นั้นครอบคลุม edge case ดีพอ คุณภาพ test ยังขึ้นกับคนเขียน/รีวิว ยังต้องมี human review อยู่ดี โดยเฉพาะ logic ที่ซับซ้อนหรือ security-sensitive

**Q: ถ้า agent ตัดสินใจผิดว่าไม่ใช่ blocker แล้วโค้ดผิดไปเลยล่ะ?**
A: เป็นความเสี่ยงจริงที่ยังมีอยู่ — blocker rule เป็น heuristic ไม่ใช่การรับประกัน จุดที่ปลอดภัยกว่าคือยังต้องมี code review ก่อน merge เสมอ โดยเฉพาะงานที่แตะ schema, auth, หรือ payment flow

**Q: ใช้กับ production code ได้จริงไหม หรือแค่ demo?**
A: ใช้ได้จริง แต่มีเงื่อนไข: ต้องมีคนดูแล CODEBASE.md ให้ทันสมัย และยังต้องมี human review ก่อน merge ไม่ใช่ตัวแทน senior review ทั้งหมด เป็นตัวช่วยลด manual work ในขั้นตอนต้นๆ (scan, AC, plan, test skeleton) มากกว่า

**Q: ทำไมไม่ใช้ TDD ไปเลย?**
A: คนละจุดประสงค์ — TDD ใช้ test นำ design ก่อนเขียนโค้ด ส่วนนี่คือ AC-driven: implement ก่อน แล้วเขียน test คู่กันเพื่อ verify กับ AC ทีหลัง เลือกใช้ AC-driven เพราะ workflow จริงส่วนใหญ่เริ่มจาก requirement ที่ยังไม่นิ่งพอจะเขียน test ก่อนได้เสมอ

**Q: repo ใหญ่มาก (mono-repo หลาย module) จะช้าไหม?**
A: ออกแบบรองรับไว้แล้ว — ถ้า repo ใหญ่ split เป็นหลายไฟล์ (`summary-auth.md`, `summary-payment.md` ฯลฯ) แล้ว agent โหลดเฉพาะไฟล์ที่เกี่ยว และ Update mode ใช้ diff จาก commit เดิมแทนการ scan ใหม่ทั้งหมด

**Q: ไฟล์ `.task/*.md` เยอะขึ้นเรื่อยๆ ใครลบ เก็บที่ไหน?**
A: ไม่ auto-delete โดยตั้งใจ — เพื่อให้ user ยังเปิดดูได้ว่า AC ไหน verify แล้วบ้าง การลบเป็นหน้าที่ user ทำเองหลัง verify เสร็จ ทีมควรตกลง convention ร่วมกัน เช่น ใส่ `.task/` ไว้ใน `.gitignore` หรือมี cleanup routine

**Q: เพิ่ม token/เวลาใช้งานไหมเทียบกับสั่งงานปกติ?**
A: มี overhead เพิ่มจาก AC extraction และเขียนไฟล์ แต่ compensate ด้วยการไม่ต้อง scan repo ใหม่ทุกครั้ง (ผ่าน CODEBASE.md) โดยรวมงานเดี่ยวๆ อาจช้ากว่าเล็กน้อย แต่งานสะสมหลายรอบใน repo เดียวกันจะเร็วขึ้น

---

## ปิดท้าย

> อะไรจะอยู่ในห้องเครื่องไม่สำคัญหรอก...
> สิ่งเดียวที่สำคัญที่สุดคือ "ใครอยู่หลังพวงมาลัย"
>
> *It doesn't matter what's under the hood...*
> *The only thing that matters is who's behind the wheel.*

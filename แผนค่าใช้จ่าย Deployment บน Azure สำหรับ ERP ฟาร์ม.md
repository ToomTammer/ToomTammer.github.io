# แผนค่าใช้จ่าย Deployment บน Azure สำหรับ ERP ฟาร์ม

บริบท: ตัดสินใจแล้วว่า deploy แบบ **Cloud เต็มรูปแบบบน Azure** (ทั้ง app server และ database) — ไม่ใช่ on-prem ตามที่ไฟล์ "การเลือก Backend Framework สำหรับ ERP ฟาร์ม.md" เคยตั้งสมมติฐานไว้ตอนแรก และฐานข้อมูลที่โปรเจกต์กำหนดคือ **PostgreSQL** (ไม่ใช่ SQL Server) ดังนั้นใช้บริการ **Azure Database for PostgreSQL** ไม่ใช่ Azure SQL Database

## ตารางค่าใช้จ่ายเริ่มต้น (ระดับ MVP / ผู้ใช้น้อย)

| # | รายการ | บริการที่ใช้ | ระดับเริ่มต้น | บาท/เดือน |
|---|---|---|---|---|
| 1 | Hosting (Web API + MVC ในแอปเดียว) | Azure App Service | B1 Basic | ~460 |
| 2 | ฐานข้อมูล | **Azure Database for PostgreSQL — Flexible Server** | Burstable **B1ms** (1 vCore, 2GB RAM) | ต้องเช็คราคาจริงที่ Azure Pricing Calculator ณ เวลา deploy (โมเดลราคาคิดแยก compute + storage ไม่ใช่แบบ DTU เหมือน SQL Database เดิม คร่าวๆ ใกล้เคียงหรือถูกกว่าที่เคยประเมินไว้ที่ ~515 สำหรับ SQL S0) |
| 3 | ไฟล์ (QR/ใบรับรอง/PDF/ไฟล์แนบ) | Azure Blob Storage | Hot tier | ~105-175 |
| 4 | รายงาน/แดชบอร์ด | Chart.js/ApexCharts ในแอปเดิม | — | 0 |
| 5 | CI/CD | GitHub Actions | Free tier | 0 |
| 6 | แจกแอปภาคสนาม | Android APK ตรง (เครื่องบริษัทแจก ไม่ผ่าน Google Play) | — | 0 |

**รวมค่าใช้จ่ายเริ่มต้นโดยประมาณ**: ใกล้เคียงเดิมที่เคยประเมินไว้ (~1,080-1,150 บาท/เดือน) แต่ต้องยืนยันตัวเลขแถวที่ 2 ใหม่จาก Azure Pricing Calculator ก่อนสรุปยอดจริง เพราะเปลี่ยนบริการฐานข้อมูล — ไม่รวมค่าเครื่อง Android ที่บริษัทจัดซื้อแจกพนักงาน (ค่าใช้จ่ายครั้งเดียว)

## แผนขยายตามจำนวนผู้ใช้

| ระดับผู้ใช้ | Hosting | ฐานข้อมูล | หมายเหตุ |
|---|---|---|---|
| 30-60 คน | App Service S1 | PostgreSQL Flexible Server ระดับ General Purpose ถัดไป (เช่น D2ds_v4) | อัปเกรดเมื่อเริ่มมี concurrent user เยอะขึ้น |
| 100+ คน หรือขยายหลายฟาร์ม | App Service P1V3 | PostgreSQL Flexible Server ระดับสูงขึ้นตามโหลดจริง | ควร monitor CPU/connection ก่อนตัดสินใจอัปเกรด ไม่อัปเกรดล่วงหน้าเกินจำเป็น |

*ตัวเลขบาท/เดือนของแต่ละระดับต้องคำนวณใหม่จาก Azure Pricing Calculator ตอนถึงเวลาจริง เพราะราคาเปลี่ยนบ่อยและต่างกันตาม region*

## แผนสำรอง: ย้ายกลับ On-Premise ในอนาคต

หากนโยบายบริษัทเปลี่ยนในอนาคตและต้องการย้ายกลับมารันเองที่ office/ฟาร์ม:

- **PostgreSQL เป็น open source เต็มรูปแบบ ไม่มีข้อจำกัดขนาดฐานข้อมูลแบบ license ฟรี** (ต่างจาก SQL Server Express ที่จำกัด 10GB ต่อฐานข้อมูล) — ย้ายกลับ on-prem ได้โดยไม่ต้องกังวลเรื่องเพดานขนาดข้อมูลหรือค่า license เพิ่มเติมเมื่อข้อมูลโตเกินจำกัด ถือเป็นข้อดีของการเลือก PostgreSQL ตั้งแต่แรกเทียบกับแผนเดิมที่เคยพิจารณา SQL Server Express
- ย้ายข้อมูลได้ตรงไปตรงมาด้วย `pg_dump`/`pg_restore` มาตรฐาน ไม่ต้องพึ่งเครื่องมือเฉพาะของ Azure

## ผลต่อการเลือก Backend Framework

เนื่องจากฐานข้อมูลกำหนดเป็น PostgreSQL แน่นอนแล้ว (ไม่ใช่ตัวเลือกเปิด) ทำให้:
- **Django** ยังคงเป็นตัวเลือกที่จับคู่ลื่นไหลที่สุด — Django ORM + psycopg เป็นคู่มาตรฐานของฝั่ง Python และยังได้ประโยชน์จาก admin panel auto-gen ที่ช่วยลดงาน CRUD 25+ โมดูลตามที่เปรียบเทียบไว้ในไฟล์ "การเลือก Backend Framework สำหรับ ERP ฟาร์ม.md"
- **.NET + EF Core** ยังใช้ PostgreSQL ได้ผ่าน Npgsql provider แต่ไม่ใช่คู่ default ที่ EF Core ออกแบบมาให้ตั้งแต่ต้น (default คือ SQL Server) — ยังเลือกได้ถ้าทีมถนัด C# มากกว่า แต่จะไม่ได้ integration ที่แนบเนียนเท่า SQL Server บน Azure
- **NestJS** (Prisma/TypeORM) ใช้ PostgreSQL ได้ดีเช่นกัน เป็นตัวเลือกรองถ้าทีม frontend ใช้ TypeScript อยู่แล้ว

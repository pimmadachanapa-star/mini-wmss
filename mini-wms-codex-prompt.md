# Codex Development Prompt: Mini-WMS (SmartStock Logistics Co., Ltd.)

## บริบทโปรเจกต์ (Project Context)

พัฒนาโปรแกรม **Warehouse Management System (WMS) ขนาดเล็ก** สำหรับบริษัทสมมติ
**"SmartStock Logistics Co., Ltd."** ซึ่งให้บริการรับ–จัดเก็บ–จัดส่งสินค้า

**เป้าหมาย:** ระบบต้องมีระเบียบ (structured) แต่เรียบง่าย เข้าใจง่าย เหมาะกับงานส่งอาจารย์/เดโมขนาดเล็ก
ไม่ต้องรองรับ Barcode/RFID จริง (จำลองด้วยการพิมพ์รหัสสินค้าแทนได้)

**Tech Stack:**
- Backend: Node.js + Express
- Database: SQLite (ไฟล์เดียว ใช้ไลบรารี `better-sqlite3` หรือ `sqlite3`)
- Frontend: HTML + Vanilla JavaScript (หรือ CLI เป็นทางเลือก)
- ไม่ต้องใช้ framework หนักๆ เช่น React ในเวอร์ชันแรก

**หลักการทำงานร่วมกับ Codex:**
- ทำทีละเฟส เฟสละ 1 ชุดคำสั่ง อย่ารวมหลายเฟสในครั้งเดียว
- แต่ละเฟสต้องรันได้จริงและทดสอบได้ก่อนไปเฟสถัดไป
- เขียนคอมเมนต์ภาษาไทยกำกับ logic สำคัญ เพื่อให้อ่านเข้าใจง่าย
- ใช้โครงสร้างไฟล์ที่ชัดเจน แยก routes / models / controllers

---

## Phase 0: Project Scaffolding (โครงสร้างเริ่มต้น)

**คำสั่งสำหรับ Codex:**
```
สร้างโครงสร้างโปรเจกต์ Node.js สำหรับระบบ Mini-WMS ดังนี้:

mini-wms/
├── package.json
├── server.js
├── db/
│   ├── database.js       (เชื่อมต่อ SQLite และสร้างตารางเริ่มต้น)
│   └── wms.sqlite        (ไฟล์ฐานข้อมูล จะถูกสร้างอัตโนมัติ)
├── models/
│   ├── product.js
│   ├── location.js
│   ├── inventory.js
│   └── order.js
├── routes/
│   ├── products.js
│   ├── locations.js
│   ├── inventory.js
│   └── orders.js
├── public/
│   ├── index.html
│   ├── style.css
│   └── app.js
└── README.md

ใช้ dependencies: express, better-sqlite3, cors, body-parser
สร้างตารางฐานข้อมูลตามโครงสร้างนี้:

1. products (id, sku, name, unit, reorder_point, created_at)
2. locations (id, location_code, zone, status) -- status: 'ว่าง' หรือ 'เต็ม'
3. inventory (id, product_id, location_id, quantity, updated_at)
4. inventory_transactions (id, product_id, type, quantity, ref_document, created_at)
   -- type: 'รับเข้า' | 'จ่ายออก' | 'คืนสินค้า'
5. orders (id, order_no, status, created_at)
   -- status: 'รอดำเนินการ' | 'จัดเตรียมแล้ว' | 'จัดส่งแล้ว'
6. order_items (id, order_id, product_id, quantity)

ให้ server.js รัน Express server บน port 3000 พร้อม endpoint ทดสอบ GET /health
ที่ตอบ { status: "ok" }
```

**เกณฑ์ตรวจสอบ (Definition of Done):**
- [ ] รัน `npm install && npm start` แล้วเซิร์ฟเวอร์ทำงานได้
- [ ] เรียก `GET /health` ได้ผลลัพธ์ถูกต้อง
- [ ] ไฟล์ฐานข้อมูล `wms.sqlite` ถูกสร้างพร้อมตารางครบ 6 ตาราง

---

## Phase 1: Receiving & Put-away (รับสินค้า + จัดเก็บ)

**คำสั่งสำหรับ Codex:**
```
พัฒนาโมดูลรับสินค้าและจัดเก็บสินค้า (Receiving & Put-away) ดังนี้:

1. Model `product.js`: ฟังก์ชัน CRUD พื้นฐาน (create, getAll, getById, update, delete)
2. Model `location.js`: ฟังก์ชัน CRUD + getAvailableLocation()
   -- getAvailableLocation() ค้นหาตำแหน่งที่ status = 'ว่าง' ตำแหน่งแรกที่เจอ
3. Route `POST /api/products` -- เพิ่มสินค้าใหม่เข้าระบบ
4. Route `POST /api/receiving` -- รับสินค้าเข้าคลัง
   Body: { product_id, quantity, ref_document }
   Logic:
   a. ค้นหาตำแหน่งว่างด้วย getAvailableLocation()
   b. ถ้าไม่มีตำแหน่งว่าง ให้ตอบ error "ไม่มีพื้นที่จัดเก็บว่าง"
   c. ถ้ามี ให้บันทึกลงตาราง inventory (เพิ่ม quantity ถ้ามีอยู่แล้วในตำแหน่งเดิม)
   d. บันทึก inventory_transactions ประเภท 'รับเข้า'
   e. อัปเดตสถานะ location เป็น 'เต็ม' ถ้าจัดเก็บเต็มพื้นที่ที่กำหนด (สมมติ 1 location เก็บได้ไม่จำกัด สำหรับเวอร์ชันนี้ให้ status = 'เต็ม' เมื่อมีสินค้าอยู่ในตำแหน่งนั้นแล้ว)
5. สร้างหน้า UI พื้นฐานใน public/index.html สำหรับ:
   - ฟอร์มเพิ่มสินค้าใหม่
   - ฟอร์มรับสินค้าเข้าคลัง (เลือกสินค้า + จำนวน)
   - ตารางแสดงรายการสินค้าและตำแหน่งจัดเก็บปัจจุบัน
```

**เกณฑ์ตรวจสอบ:**
- [ ] เพิ่มสินค้าใหม่ผ่าน API/UI ได้
- [ ] รับสินค้าเข้าคลังแล้วระบบเลือกตำแหน่งจัดเก็บอัตโนมัติ
- [ ] มีบันทึกใน inventory_transactions ทุกครั้งที่รับสินค้า

---

## Phase 2: Order Management & Picking (จัดการคำสั่งซื้อ + จัดเตรียมสินค้า)

**คำสั่งสำหรับ Codex:**
```
พัฒนาโมดูลจัดการคำสั่งซื้อและ Picking ดังนี้:

1. Model `order.js`: create order พร้อม order_items, getOrderById (join กับ order_items)
2. Route `POST /api/orders` -- สร้างคำสั่งซื้อใหม่
   Body: { items: [{ product_id, quantity }, ...] }
   Logic: สร้าง order status 'รอดำเนินการ' พร้อมบันทึก order_items
3. Route `POST /api/orders/:id/pick` -- จัดเตรียมสินค้าตามคำสั่งซื้อ
   Logic:
   a. ตรวจสอบสต็อกคงเหลือของทุกสินค้าใน order เพียงพอหรือไม่
   b. ถ้าไม่พอ ตอบ error ระบุสินค้าที่ขาด พร้อมจำนวนที่ขาด
   c. ถ้าพอ ให้ตัดสต็อกจาก inventory (ลด quantity)
   d. บันทึก inventory_transactions ประเภท 'จ่ายออก'
   e. อัปเดต order status เป็น 'จัดเตรียมแล้ว'
   f. ถ้าตำแหน่งใดสินค้าหมด (quantity = 0) ให้อัปเดต location status กลับเป็น 'ว่าง'
4. Route `POST /api/orders/:id/ship` -- เปลี่ยนสถานะเป็น 'จัดส่งแล้ว'
5. เพิ่ม UI ส่วนสร้างคำสั่งซื้อ และปุ่มดำเนินการ Pick / Ship พร้อมแสดงสถานะ order
```

**เกณฑ์ตรวจสอบ:**
- [ ] สร้างคำสั่งซื้อได้และตรวจสอบสต็อกก่อนตัดจริง
- [ ] กรณีสต็อกไม่พอ ระบบแจ้งเตือนชัดเจน ไม่ตัดสต็อกผิดพลาด
- [ ] สถานะคำสั่งซื้อเปลี่ยนตามลำดับ รอดำเนินการ → จัดเตรียมแล้ว → จัดส่งแล้ว

---

## Phase 3: Inventory Dashboard & KPI Analytics (แดชบอร์ดและตัวชี้วัด)

**คำสั่งสำหรับ Codex:**
```
พัฒนาโมดูล Dashboard และ KPI พื้นฐาน อ้างอิงตัวชี้วัดจากเอกสารบริษัท:

1. Route `GET /api/dashboard/summary` ส่งข้อมูลสรุป:
   - จำนวนสินค้าทั้งหมด (total_products)
   - จำนวนตำแหน่งจัดเก็บที่ใช้งาน / ว่าง (location_usage_percent)
   - รายการสินค้าที่ต่ำกว่าจุดสั่งซื้อ (reorder_point) -- low_stock_items
   - จำนวนคำสั่งซื้อแยกตามสถานะ (orders_by_status)
   - จำนวนรายการรับเข้า/จ่ายออกในวันนี้ (today_transactions)

2. คำนวณ "การใช้พื้นที่คลังสินค้า" = (จำนวน location ที่ status='เต็ม' / จำนวน location ทั้งหมด) * 100

3. สร้างหน้า Dashboard ใน public/index.html หรือหน้าใหม่ dashboard.html แสดง:
   - การ์ดตัวเลขสรุป (total_products, location_usage_percent)
   - ตารางสินค้าที่ใกล้หมด (low_stock_items) เน้นสีแดง/เหลืองเพื่อเตือน
   - กราฟแท่งอย่างง่าย (ใช้ Chart.js ผ่าน CDN) แสดงจำนวน order แยกตามสถานะ
```

**เกณฑ์ตรวจสอบ:**
- [ ] Dashboard แสดงข้อมูลจริงจากฐานข้อมูล ไม่ hardcode
- [ ] สินค้าที่ต่ำกว่าจุดสั่งซื้อแสดงผลถูกต้อง
- [ ] เปอร์เซ็นต์การใช้พื้นที่คำนวณถูกต้องตามสูตร

---

## Phase 4 (ทางเลือกเสริม): REST API Layer สมบูรณ์ + เอกสารประกอบ

**คำสั่งสำหรับ Codex:**
```
ปรับปรุงระบบให้เป็น REST API ที่สมบูรณ์และมีเอกสารประกอบ:

1. เพิ่ม middleware error handling กลาง (express error handler)
   ตอบกลับรูปแบบมาตรฐาน: { success: boolean, data | error }
2. เพิ่ม input validation เบื้องต้น (เช่น ใช้ express-validator หรือเขียนเช็คเอง)
   ตรวจสอบ: quantity ต้อง > 0, product_id ต้องมีอยู่จริง ฯลฯ
3. เพิ่ม endpoint สำหรับการคืนสินค้า (Return):
   POST /api/returns -- { product_id, quantity, reason }
   บันทึก inventory_transactions ประเภท 'คืนสินค้า' และเพิ่มสต็อกกลับเข้า inventory
4. เขียน README.md อธิบาย:
   - วิธีติดตั้งและรันโปรเจกต์
   - รายการ API endpoints ทั้งหมดพร้อมตัวอย่าง request/response
   - ผังงาน (workflow) ของระบบแบบข้อความ (รับ → จัดเก็บ → คำสั่งซื้อ → จัดเตรียม → จัดส่ง → คืน)
```

**เกณฑ์ตรวจสอบ:**
- [ ] ทุก endpoint ตอบกลับในรูปแบบมาตรฐานเดียวกัน
- [ ] มี validation ป้องกัน input ผิดพลาดเบื้องต้น
- [ ] README.md อ่านแล้วสามารถติดตั้งและใช้งานระบบได้ทันที

---

## หมายเหตุการใช้งานร่วมกับ Codex

1. คัดลอกคำสั่งในแต่ละ Phase ไปวางใน Codex ทีละเฟส
2. หลังรันเสร็จแต่ละเฟส ให้ตรวจสอบตาม "เกณฑ์ตรวจสอบ" ก่อนไปเฟสถัดไป
3. หากต้องการปรับ scope ให้เล็กลงอีก สามารถข้าม Phase 4 ได้ ระบบ Phase 0-3
   ก็ครอบคลุมวงจรหลักของ WMS ตามเอกสาร (รับ → จัดเก็บ → คำสั่งซื้อ → จัดเตรียม → จัดส่ง) แล้ว
4. Phase 3 (Dashboard/KPI) อ้างอิงตัวชี้วัดจริงจากเอกสารบริษัท SmartStock Logistics
   (เช่น ความแม่นยำการสั่งซื้อ, การใช้พื้นที่คลัง) สามารถเพิ่มตัวชี้วัดอื่นได้ตามต้องการ

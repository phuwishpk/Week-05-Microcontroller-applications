# 📊 รายงานสรุปผลการทดลอง: การศึกษาสถาปัตยกรรม ESP32

---

## 🧑‍🎓 ข้อมูลนักศึกษา

- **ชื่อ:** ภูวิชญ์ ประกอบจิตร
- **รหัสนักศึกษา:** 67030183
- **วันที่ทำการทดลอง:** 10-8-2025

---

## ✅ Checklist การทดลอง

- [x] Environment setup สำเร็จ (ต่อเนื่องจากสัปดาห์ที่ 4)
- [x] Memory architecture analysis เสร็จสมบูรณ์
- [x] Cache performance testing เสร็จสมบูรณ์
- [x] Dual-core analysis เสร็จสมบูรณ์
- [x] รายงานผลการทดลองครบถ้วน

---

## 🔬 ผลการทดลองที่ 2: การศึกษา Memory Architecture

### สรุปผลการทดลอง
จากการรันโปรแกรม `memory_test.c` สามารถระบุตำแหน่งของตัวแปรประเภทต่างๆ ในหน่วยความจำได้ดังนี้:

**Table 2.1: Memory Address Analysis**
| Memory Section | Variable/Function | Address (ที่แสดงออกมา) | Memory Type |
|----------------|-------------------|----------------------|-------------|
| Stack | stack_var | 0x3ffb4550 | SRAM |
| Global SRAM | sram_buffer | 0x3ffb16ac | SRAM |
| Flash | flash_string | 0x3f407b64 | Flash |
| Heap | heap_ptr | 0x3ffb5264 | SRAM |

**Table 2.2: Memory Usage Summary**
| Memory Type | Free Size (bytes) | Total Size (bytes) |
|-------------|-------------------|--------------------|
| Internal SRAM | 380096 byte | 520,192 |
| Flash Memory | 0 bytes | varies |
| DMA Memory | 303096 bytes | varies |

### คำถามวิเคราะห์
1.  **Memory Types**: **SRAM** (Static RAM) ใช้สำหรับเก็บข้อมูลที่เปลี่ยนแปลงตลอดเวลาและต้องการการเข้าถึงที่รวดเร็ว เช่น Stack (ตัวแปร local), Heap (การจองหน่วยความจำแบบ dynamic), และตัวแปร Global (DRAM) **Flash Memory** ใช้สำหรับเก็บข้อมูลที่ถาวรและไม่เปลี่ยนแปลงบ่อย เช่น โค้ดโปรแกรม (Instructions) และข้อมูลคงที่ (Constants)
2.  **Address Ranges**: จากผลการทดลอง ตัวแปรที่อยู่ใน SRAM (Stack, Global SRAM, Heap) จะมี Address อยู่ในย่าน `0x3ffbxxxx` ส่วนข้อมูลที่มาจาก Flash (flash_string) จะถูก map มาที่ Address ย่าน `0x3f40xxxx` (หรือ `0x4xxxxxxx` ผ่าน Cache)
3.  **Memory Usage**: จาก Table 2.2, ESP32 มี Internal SRAM ทั้งหมด 520,192 bytes และในขณะที่ทดสอบ มีหน่วยความจำว่าง (Free heap) เหลือ 380,096 bytes

---

## 🔬 ผลการทดลองที่ 3: การศึกษา Cache Performance

### สรุปผลการทดลอง
จากการเปรียบเทียบเวลาในการเข้าถึงหน่วยความจำในรูปแบบต่างๆ ได้ผลดังนี้:

**Table 3.1: Cache Performance Results**
| Test Type | Memory Type | Time (μs) | Ratio vs Sequential |
|-----------|-------------|-----------|-------------------|
| Sequential | Internal SRAM | 10089 | 1.00x |
| Random | Internal SRAM | 13599 | 1.35x |
| Sequential | External Memory | 49897 | 4.95x (เทียบ SRAM) |
| Random | External Memory | 37140 | 3.68x |

**Table 3.2: Stride Access Performance (Internal SRAM)**
| Stride Size | Time (μs) | Ratio vs Stride 1 |
|-------------|-----------|------------------|
| 1 | 6350 | 1.00x |
| 2 | 3373 | 0.53x |
| 4 | 1631 | 0.26x |
| 8 | 862 | 0.14x |
| 16 | 1199 | 0.19x |

### คำถามวิเคราะห์
1.  **Cache Efficiency**: การเข้าถึงแบบ **Sequential** (10089 μs) เร็วกว่าแบบ **Random** (13599 μs) เนื่องจาก CPU จะดึงข้อมูลจาก Memory มาเก็บใน Cache เป็นบล็อก (Cache Line) การเข้าถึงข้อมูลแบบเรียงลำดับจึงมีโอกาสสูงที่จะเกิด **Cache Hit** (ข้อมูลอยู่ใน Cache แล้ว) ในขณะที่การเข้าถึงแบบสุ่มจะทำให้เกิด **Cache Miss** บ่อยครั้ง ทำให้ CPU ต้องไปดึงข้อมูลจาก SRAM ที่ช้ากว่ามาใหม่
2.  **Memory Hierarchy**: **Internal SRAM** เร็วกว่า **External Memory** อย่างมาก (Sequential: 10089 μs vs 49897 μs) เพราะ Internal SRAM เชื่อมต่อกับ CPU โดยตรงด้วยบัสความเร็วสูง ในขณะที่ External Memory (PSRAM) ต้องเข้าถึงผ่าน SPI bus ซึ่งมีความเร็วต่ำกว่าและมี Latency สูงกว่า
3.  **Stride Patterns**: เมื่อ Stride เพิ่มขึ้น (1, 2, 4, 8) เวลาที่ใช้จะน้อยลง เพราะจำนวนครั้งในการเข้าถึงข้อมูลทั้งหมดใน Loop ลดลง (เช่น Stride 4 เข้าถึงข้อมูลเพียง 1/4 ของทั้งหมด) อย่างไรก็ตาม ที่ Stride 16 เวลาเพิ่มขึ้นเล็กน้อย อาจเกิดจากรูปแบบการเข้าถึงที่ไม่พอดีกับขนาดของ Cache Line ทำให้เกิด Cache Miss มากขึ้นเมื่อเทียบกับ Stride 8

---

## 🔬 ผลการทดลองที่ 4: การศึกษา Dual-Core Architecture

### สรุปผลการทดลอง
การทดสอบรัน Task แยกกันบน 2 Core (PRO_CPU และ APP_CPU) และสื่อสารกันผ่าน Queue ได้ผลดังนี้:

**Table 4.1: Dual-Core Performance Summary**
| Metric | Core 0 (PRO_CPU) | Core 1 (APP_CPU) |
|--------|-------------------|-------------------|
| Total Iterations | 81 | 101 |
| Average Time per Iteration (μs) | 115 | 10176 |
| Total Execution Time (ms) | 4997 | 4997 |
| Task Completion Rate | 100% | 100% |

**Table 4.2: Inter-Core Communication**
| Metric | Value |
|--------|-------|
| Messages Sent | 90 |
| Messages Received | 9 |
| Average Latency (μs) | 16957 |
| Queue Overflow Count | 0 |

### คำถามวิเคราะห์
1.  **Core Specialization**: จากผลลัพธ์ **Core 0 (PRO_CPU)** ใช้เวลาเฉลี่ยต่อรอบเพียง **115 μs** ซึ่งเหมาะกับงานที่รวดเร็วและตอบสนองทันที (เช่น การจัดการ Protocol Stack) ในขณะที่ **Core 1 (APP_CPU)** ใช้เวลาเฉลี่ยสูงถึง **10176 μs** (เนื่องจากมีการคำนวณ Floating point `sqrt`) จึงเหมาะสำหรับงานประมวลผลหลักของ Application ที่ซับซ้อน
2.  **Communication Overhead**: การสื่อสารระหว่าง Core มี Overhead (Latency) เฉลี่ยสูงถึง **16,957 μs** (ประมาณ 17 ms) นี่คือต้นทุนของการส่งข้อมูลข้าม Core ซึ่งแสดงให้เห็นว่าการสื่อสารนี้ไม่ใช่งานที่เบา และต้องออกแบบระบบให้ดีหากต้องการการตอบสนองแบบ Real-time
3.  **Load Balancing**: การทดลองนี้แสดงการ **Task Isolation** (การแยกงาน) มากกว่าการ Load Balancing Core ทั้งสองทำงานของตัวเองไปพร้อมกัน (Parallel Processing) แม้ว่างานของ Core 1 จะหนักกว่า (10176 μs/iter) แต่งานของ Core 0 (115 μs/iter) ก็ไม่ถูกรบกวน นี่คือข้อดีหลักของสถาปัตยกรรม Dual-Core ใน ESP32

---

## 💬 คำถามเพิ่มเติม

1.  **เปรียบเทียบประสบการณ์การใช้ Docker ในสัปดาห์นี้กับสัปดาห์ที่ 4:**
    การใช้ Docker ในสัปดาห์นี้ให้ความรู้สึกคุ้นเคยและสะดวกสบายมาก เนื่องจากใช้ `docker-compose.yml` และคำสั่ง `docker-compose up -d`, `docker-compose exec` แบบเดิมจากสัปดาห์ที่ 4 ทำให้ไม่ต้องเสียเวลาเรียนรู้การตั้งค่า Environment ใหม่ สามารถเริ่มทำการทดลองและโฟกัสกับเนื้อหาเกี่ยวกับสถาปัตยกรรม ESP32 ได้ทันที

2.  **สิ่งที่เรียนรู้เพิ่มเติมเกี่ยวกับ ESP32 architecture:**
    ได้เรียนรู้และเห็นภาพการทำงานจริงของสถาปัตยกรรม ESP32 ใน 3 ประเด็นหลัก:
    * **Memory Layout:** เข้าใจการแบ่งพื้นที่หน่วยความจำว่าส่วนไหน (SRAM, Flash) เก็บข้อมูลประเภทใด (Stack, Heap, Code) และตรวจสอบ Address จริงได้อย่างไร
    * **Cache System:** เห็นผลกระทบของ Cache ต่อประสิทธิภาพอย่างชัดเจน ว่าการเขียนโค้ดโดยคำนึงถึง locality (Sequential Access) ช่วยให้โปรแกรมเร็วกว่าการเข้าถึงแบบสุ่ม (Random Access)
    * **Dual-Core:** เข้าใจแนวคิดการแบ่งงานระหว่าง PRO_CPU (สำหรับงาน Protocol) และ APP_CPU (สำหรับ Application) และตระหนักถึงต้นทุน (Overhead) ที่เกิดขึ้นจริงจากการสื่อสารระหว่าง Core

3.  **ความท้าทายที่พบในการทำ architecture analysis:**
    ความท้าทายหลักคือการ "ตีความ" ผลลัพธ์ที่ได้จากการทดลอง ตัวเลขเวลา (μs) หรือ Address ที่แสดงออกมา ไม่ได้บอกอะไรในตัวเอง แต่ต้องอาศัยความรู้จากเอกสาร `ESP32_Architecture.md` มาวิเคราะห์ประกอบ เช่น การวิเคราะห์ว่าทำไม Latency ถึงสูง (16957 μs) หรือทำไม Stride 16 ถึงช้ากว่า Stride 8 ซึ่งต้องใช้การคิดวิเคราะห์เชื่อมโยงระหว่างผลลัพธ์กับทฤษฎีสถาปัตยกรรม
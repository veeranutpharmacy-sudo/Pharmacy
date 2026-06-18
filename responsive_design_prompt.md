# Prompt: Responsive Design — Interactive Hospital Map

## บริบท (Context)

เว็บไซต์เป็น **Interactive Hospital Map** — แผนที่โรงพยาบาลแบบ Single Page HTML ที่มีองค์ประกอบดังนี้:

- **Map viewport**: แผนที่ full-screen ที่ซูมเข้า-ออกได้ด้วย CSS `transform: scale()`
- **Building overlays**: รูป PNG/WebP ซ้อนบนแผนที่ แสดงตึกแต่ละหลัง
- **Callout boxes**: กล่องข้อความลอยชี้ชื่อตึก มีเส้น callout-line ต่อลงมา
- **SVG hotspot polygons**: พื้นที่คลิกได้บนแผนที่
- **Modal (Layer 1)**: เมื่อคลิกตึก → ซูมเข้า → แสดง modal เต็มจอ มีการ์ดวิดีโอ YouTube เรียงเป็น grid 3 คอลัมน์
- **Detail view (Layer 2)**: คลิกการ์ด → แสดงวิดีโอขนาดใหญ่ + รายละเอียด + ปุ่ม prev/next
- **Hint bar**: แถบแนะนำด้านล่าง
- **License text**: ข้อความลิขสิทธิ์มุมขวาล่าง
- **Loading screen**: หน้าจอโหลดตอนเปิดเว็บ

เดิมออกแบบมาสำหรับ **Desktop เท่านั้น** (1920×1080+) ต้องการทำให้รองรับทุกขนาดจอ

---

## สิ่งที่ต้องทำ (Requirements)

### 1. CSS Media Queries — 5 Breakpoints

#### Tablet (≤1024px)
- Modal grid: 3 คอลัมน์ → **2 คอลัมน์**
- Callout boxes: ลด padding จาก 16px 24px → 12px 18px, border-radius 12px
- Callout title: font-size 24px → 18px
- Modal title ไทย: 48px → 40px
- Modal title อังกฤษ: 24px → 20px

#### Small Tablet / Large Phone (≤768px)
- **Callout line**: ซ่อน (`display: none`)
- **Callout box**: แสดงเป็น **pill label** ขนาดเล็ก
  - `border-radius: 20px` (pill shape)
  - `border: 1.5px solid rgba(59, 130, 246, 0.5)` (ขอบฟ้า)
  - `background: rgba(255, 255, 255, 0.92)` (พื้นขาวทึบ)
  - `backdrop-filter: none` (ปิด blur เพื่อ performance)
  - `box-shadow: 0 2px 10px rgba(0,0,0,0.12), 0 0 0 1px rgba(255,255,255,0.5) inset` (เงา + inner glow)
  - `margin-top: -16px !important` (override inline style ให้ชิดตึก)
  - Callout title: font 11px, สีน้ำเงินเข้ม `#1e40af`, font-weight 600
- **Modal**: padding 16px, padding-top คำนวณจาก safe-area
- **Modal grid**: 1 คอลัมน์
- **Modal title**: ไทย 28px, อังกฤษ 14px (opacity 0.75)
- **ปุ่มปิด Modal**: pill shape (`border-radius: 20px`, `padding: 10px 18px`, `background: rgba(255,255,255,0.85)`)
- **ปุ่มปิด Detail**: วงกลม (`border-radius: 50%`, 44×44px)
- **Detail container**: width 95%, max-height คำนวณจาก safe-area
- **Detail video**: border-radius 14px, box-shadow เข้มขึ้น
- **Detail nav buttons**: padding 12px 20px, min-width 100px, border-radius 14px
- **Hint**: max-width 88vw, text-align center, border-radius 20px
- **License**: ย้ายมากลางล่าง (left: 8px, right: 8px, text-align: center)

#### Phone (≤480px)
- **Intro title**: 26px, filter drop-shadow ขาวเพื่อให้อ่านง่าย
- **Modal**: padding 12px 8px
- **Modal title**: ไทย 24px, อังกฤษ 13px
- **Content card**: border-radius 14px
- **Card text**: padding 12px 14px, title 13.5px, desc 11px (line-clamp 3)
- **Detail**: width 100%, padding 0 10px
- **Detail video**: border-radius 10px, play button 44px
- **Callout box**: padding 5px 10px, margin-top -12px, font 10px
- **Hint**: bottom เว้น safe-area + 40px

#### Very Small Phone (≤360px)
- Intro title: 22px
- Modal title: ไทย 20px, อังกฤษ 12px
- Detail title: 15px, desc: 12px

#### Landscape Phone (max-height: 500px, orientation: landscape)
- Modal grid: **2 คอลัมน์**
- Modal title: ไทย 22px, อังกฤษ 13px
- Modal header margin-bottom: 8px
- Detail container: max-height 85vh
- Detail text-box padding: 14px 18px

---

### 2. iPhone Safe Area (Notch)

ทุก element ที่ชิดขอบจอต้องใช้ `env(safe-area-inset-*)`:

```css
/* ตัวอย่าง */
padding-top: calc(56px + env(safe-area-inset-top, 0px));
bottom: calc(16px + env(safe-area-inset-bottom, 0px));
```

จุดที่ต้องใช้:
- `#modal` → padding-top, padding-bottom
- `.modal-close-btn` → top
- `.detail-close-btn` → top
- `.detail-container` → max-height
- `#hint` → bottom
- `#license-text` → bottom
- `.intro-text` → margin-bottom

---

### 3. JavaScript — Responsive Zoom Scale

แทนที่ค่า zoom scale คงที่ ให้ใช้ฟังก์ชันที่ปรับตามขนาดจอ:

```javascript
function getZoomScale() {
  const w = window.innerWidth;
  if (w <= 480)  return 1.4;  // มือถือ
  if (w <= 768)  return 1.6;  // tablet เล็ก
  if (w <= 1024) return 1.9;  // tablet
  return 2.2;                  // desktop
}
```

เรียกใช้ `getZoomScale()` แทน `ZOOM_SCALE` constant ทุกที่ที่คำนวณ transform

---

### 4. หลักการออกแบบ (Design Principles)

1. **คงรูปแบบเดิม** — แผนที่, modal grid แนวตั้ง, detail view ทุกอย่างเหมือน desktop แค่ย่อขนาด
2. **Touch-friendly** — ปุ่มทุกปุ่มต้องมี touch target ≥44px
3. **Performance-first** — ปิด `backdrop-filter` บนมือถือ, ใช้ `background` ทึบแทน
4. **Progressive reduction** — ลดความซับซ้อนตามขนาดจอ (ซ่อน callout-line, ลด line-clamp, ลด padding)
5. **Safe area aware** — รองรับ iPhone notch/Dynamic Island ทุกจุด
6. **ชื่อตึกต้องเห็น** — ไม่ซ่อน callout box บนมือถือ แค่ย่อเป็น pill label

---

### 5. ข้อควรระวัง (Constraints)

- ❌ **ห้าม** ใช้ horizontal scroll สำหรับ modal cards (ผู้ใช้ต้องการ grid แนวตั้ง)
- ❌ **ห้าม** ซ่อนชื่อตึก (callout boxes) บนมือถือ — ต้องแสดงเป็น label เสมอ
- ❌ **ห้าม** ใช้ `backdrop-filter` บน callout boxes ในจอ ≤768px
- ✅ **ใช้** `!important` เฉพาะ `margin-top` ของ callout-box เพื่อ override inline styles
- ✅ **ใช้** `env(safe-area-inset-*)` กับ fallback value เสมอ เช่น `env(safe-area-inset-top, 0px)`

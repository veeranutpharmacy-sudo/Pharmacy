# Walkthrough — Pharmacy Hospital Map

> **ไฟล์หลัก:** [index.html](file:///Users/panuwatwoeiram/Documents/GitHub/Pharmacy/index.html)
> **Production URL:** https://pharmacy-hospital-map.vercel.app

---

## 1. Code Review 🔍

รีวิวโค้ดทั้งหมดของ `index.html` พบประเด็นที่ต้องปรับปรุง:

| หมวด | ประเด็น |
|---|---|
| Performance | รูป 8K (8360×4705) ×5 ใบ ทำให้ GPU texture memory สูงมาก |
| Performance | `backdrop-filter: blur()` ทำงานระหว่าง zoom transition → jank |
| Performance | `setTimeout` ไม่ sync กับ animation จริง |
| Accessibility | ขาด keyboard navigation, focus management |
| Code Quality | ไม่มี `contain` property, ไม่มี layer management |

---

## 2. Performance Optimization — Zoom Smoothness ⚡

### ปัญหา
ซูมเข้า-ออกแล้วมี **อาการกระตุก (jank)** อย่างเห็นได้ชัด

### Root Cause
- **รูปภาพ 8K** (8360×4705) × 5 ใบ = ~196 ล้าน pixels ต้อง composite บน GPU ระหว่าง transform
- **`backdrop-filter: blur()`** ทำงานทุกเฟรมระหว่าง transition
- **Layer ซ้อนกันหลายชั้น** (building overlays แต่ละตัวเป็น compositor layer แยก)
- **`setTimeout`** ไม่ตรงกับ animation timing จริง

### วิธีแก้

#### 2.1 ลดขนาดรูปจาก 8K → 2500px

```
map-base.webp:  1,008 KB → 278 KB  (-72%)
building-1~4:   757 KB → 135 KB   (-82%)
รวม:            1,765 KB → 413 KB  (-77%)
```

GPU texture memory ลดจาก **~196M pixels → ~18M pixels** (ลด **10 เท่า**)

ใช้คำสั่ง:
```bash
cwebp -resize 2500 0 -q 80 source.webp -o output.webp
```

#### 2.2 ปิด expensive filters ระหว่างซูม

เพิ่ม CSS class `.zooming` ที่ปิด `backdrop-filter` และ flatten layers:

```css
#map-wrapper.zooming .callout-box {
  backdrop-filter: none;
  will-change: auto;
}
```

#### 2.3 ใช้ `requestAnimationFrame` + forced reflow

```javascript
mapWrapper.classList.add('zooming');
requestAnimationFrame(() => {
  mapWrapper.offsetHeight; // force layout commit
  mapWrapper.style.transform = `translate(...)  scale(...)`;
});
```

#### 2.4 เปลี่ยน `setTimeout` → `transitionend`

```javascript
mapWrapper.addEventListener('transitionend', function handler(e) {
  if (e.propertyName === 'transform') {
    mapWrapper.removeEventListener('transitionend', handler);
    mapWrapper.classList.remove('zooming');
    // ... show UI elements
  }
});
```

#### 2.5 เพิ่ม rendering hints

```css
#map-wrapper {
  backface-visibility: hidden;
  contain: layout style;
}
```

ลบ `loading="lazy"` จาก building overlays เพื่อป้องกันรูปโหลดระหว่างซูม

---

## 3. Vercel Deployment 🚀

Deploy static site ขึ้น Vercel production:

```bash
vercel --prod --yes
```

**URL:** https://pharmacy-hospital-map.vercel.app

---

## 4. Responsive Design 📱

### ปัญหา
เว็บออกแบบมาสำหรับ Desktop เท่านั้น — บนมือถือ text ล้นจอ, modal แน่นเกินไป, ปุ่มเล็กเกินไปสำหรับ touch

### วิธีแก้ — Media Queries 5 breakpoints

#### 4.1 Tablet (≤1024px)
- Modal grid: 3 คอลัมน์ → **2 คอลัมน์**
- Callout boxes: padding/font ลดลง
- Title: 48px → 40px

#### 4.2 Small Tablet (≤768px)
- Modal grid: → **1 คอลัมน์**
- Callout boxes: ซ่อน callout-line, แสดง callout-box เป็น **pill label** ขนาดเล็ก
- ปุ่มปิด Modal: เป็น **pill shape** ใหญ่ขึ้นสำหรับ touch
- ปุ่มปิด Detail: เป็น **วงกลม**
- รองรับ **iPhone safe-area-inset** (notch)
- License text: ย้ายมาตรงกลางล่าง

#### 4.3 Phone (≤480px)
- Title: 26px, card text เล็กลง
- Detail view: padding แน่นขึ้น, border-radius ลดลง
- Callout label: font 10px, padding ลดลง

#### 4.4 Very Small Phone (≤360px)
- Title/text ลดอีกระดับ

#### 4.5 Landscape Phone
- Modal grid: 2 คอลัมน์
- Padding แน่นขึ้นเพื่อใช้พื้นที่จอแนวนอน

### Responsive Zoom Scale

```javascript
function getZoomScale() {
  const w = window.innerWidth;
  if (w <= 480) return 1.4;   // มือถือ
  if (w <= 768) return 1.6;   // tablet เล็ก
  if (w <= 1024) return 1.9;  // tablet
  return 2.2;                  // desktop
}
```

ป้องกัน map ซูมหลุดออกนอกจอบนมือถือ

---

## 5. Mobile UI Polish ✨

### ปัญหา
บนมือถือ ชื่อตึกไม่ขึ้น + UI ดูไม่สวย

### วิธีแก้

#### Callout Labels — Pill Style
```css
.callout-box {
  border-radius: 20px;
  border: 1.5px solid rgba(59, 130, 246, 0.5);
  background: rgba(255, 255, 255, 0.92);
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.12),
              0 0 0 1px rgba(255,255,255,0.5) inset;
}
.callout-title {
  color: #1e40af;
  font-size: 11px;
}
```

#### iPhone Safe Area
ทุก element ที่ชิดขอบจอใช้ `env(safe-area-inset-*)`:
```css
padding-top: calc(56px + env(safe-area-inset-top, 0px));
bottom: calc(16px + env(safe-area-inset-bottom, 0px));
```

#### ปุ่มปิดที่สวยขึ้น
- **Modal close**: pill shape, `border-radius: 20px`, background ขาว 85%
- **Detail close**: วงกลม, `border-radius: 50%`, เงาชัดขึ้น

---

## สรุปไฟล์ที่แก้ไข

| ไฟล์ | การเปลี่ยนแปลง |
|---|---|
| [index.html](file:///Users/panuwatwoeiram/Documents/GitHub/Pharmacy/index.html) | CSS responsive, JS zoom optimization, rendering hints |
| `assets/images/map-base.webp` | ลดจาก 8K → 2500px |
| `assets/images/building-1~4.webp` | ลดจาก 8K → 2500px |

---

## Production

🌐 **Live:** https://pharmacy-hospital-map.vercel.app

Deploy ทั้งหมด 5+ ครั้งผ่าน `vercel --prod --yes`

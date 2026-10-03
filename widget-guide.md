# Hướng dẫn tạo Widget Desktop cho Workspace OS (WOS)

> Tài liệu chính thức quy ước cách viết widget plugin sao cho **thống nhất 100% với
> các widget mặc định** (Đồng hồ, Thời tiết, Ghi chú, CPU/RAM, Việc hôm nay, Timeline tuần).
> Widget viết đúng quy ước này sẽ tự động có: vỏ kính mờ, thanh tiêu đề kéo được,
> nút resize, lưu vị trí theo tài khoản, dark mode và xuất hiện trong **Widget Manager**
> (Settings → Appearance → Desktop & Dock) cùng menu chuột phải Desktop.

---

## 1. Hai hệ plugin — chọn hệ nào?

| | Hệ **plugin.json** (khuyến nghị) | Hệ **legacy .wos / wos-module.json** |
|---|---|---|
| Cách cài | zip / thư mục `storage/plugins/<slug>/` có `plugin.json` | zip `.wos` qua Settings → Plugins (thẻ cũ) |
| Entry | `main.js` chạy trong sandbox runtime — nhận đối tượng `WOS` làm tham số | `main.js` load qua `<script>` — gọi `window.WOS` |
| Đăng ký app | `WOS.registerApp({ component, windowSize })` | `WOS.registerApp({ id, component, ... })` |
| Đăng ký widget | `WOS.registerWidget({ type, title, icon, defaultW, defaultH, minW, minH, component })` | `WOS.registerWidget(def, slug?)` — **ĐÃ HỖ TRỢ TỪ WOS 1.1.0** |

> ⚠️ **Sửa lỗi quan trọng (WOS 1.1.0)**: trước đây `WOS.registerWidget` chỉ tồn tại
> trong hệ plugin.json — plugin viết theo hệ legacy gọi `window.WOS.registerWidget`
> sẽ nhận `undefined` → widget không bao giờ xuất hiện (kèm cảnh báo console
> *"Lõi Workspace OS chưa có WOS.registerWidget"*). Giờ **cả hai hệ** ghi vào cùng
> một registry (`src/core/client/plugin-widgets.ts`), widget hiển thị y hệt nhau.

**Kiểm tra nhanh trong DevTools console:**
```js
typeof WOS?.registerWidget   // phải trả về "function" (đăng nhập trước)
```

---

## 2. Cấu trúc gói plugin (hệ plugin.json — khuyến nghị)

```
my-widget-plugin/
├── plugin.json     # manifest bắt buộc
├── main.js        # entry — ES5, KHÔNG dùng JSX/ESM import
└── style.css      # (tuỳ chọn) style riêng — WOS.css() nạp giùm
```

**plugin.json — schema đầy đủ (quy ước bắt buộc):**

```json
{
  "slug": "my-widget-plugin",
  "name": "Tên hiển thị",
  "description": "Mô tả ngắn (≤ 300 ký tự)",
  "icon": "🧩",
  "color": "#7c3aed",
  "kind": "app",
  "version": "1.0.0",
  "author": "Tên bạn",
  "desktopIcon": true,
  "showDock": false,
  "windowSize": { "w": 360, "h": 480 },
  "entry": "main.js",
  "styles": ["style.css"]
}
```

| Trường | Quy ước |
|---|---|
| `slug` | `[a-z0-9-]`, 2–39 ký tự, **duy nhất toàn hệ thống** |
| `kind` | `"app"` (có cửa sổ + widget) hoặc `"pet"` (nhân vật) |
| `entry` | tên file entry HOẶC code JS inline |
| `icon` | emoji ≤ 12 ký tự (Lucide không truyền qua JSON runtime được) |
| `color` | `#rrggbb` — màu nhấn của app |
| `windowSize` | 280–1400 × 240–1200 px |

---

## 3. API `WOS.registerWidget` — tham số

```js
WOS.registerWidget({
  type: 'quote',        // (bắt buộc) mã loại widget — DUY NHẤT trong plugin
  title: 'Câu nói hay', // (bắt buộc) tên hiện trong Widget Manager
  icon: '💬',           // emoji ≤ 8 ký tự — hiện trên thanh tiêu đề widget
  defaultW: 300,        // ngang mặc định — kẹp 160..800
  defaultH: 220,       // cao mặc định — kẹp 120..900
  minW: 200,            // ngang tối thiểu — kẹp 140..400
  minH: 160,            // cao tối thiểu — kẹp 100..500
  component: QuoteWidget, // (bắt buộc) React component
})
```

- Type nội bộ trở thành `"<slug>:<type>"` → **không bao giờ đụng** widget hệ thống.
- Với hệ legacy: `WOS.registerWidget(def)` tự dò slug plugin đang chạy; hoặc truyền
  tường minh `WOS.registerWidget(def, 'my-plugin')`.
- Gỡ widget khi plugin bị reset: `WOS.unregisterWidgets()` (hệ plugin.json tự gọi).

Trong entry plugin.json runtime, `WOS` có sẵn (không cần `window.`):
`h` (= React.createElement), `useState/useEffect/useRef/useMemo/useCallback`,
`api(path, opts)`, `bus`, `toast`, `openApp(moduleId)`, `storage.get/set`,
`css(text)`, `assetUrl(file)`, `registerApp`, `registerPet`, `registerWidget`.

---

## 4. QUY ƯỚC STYLE — bắt buộc để "như widget mặc định"

Widget của bạn được vẽ **bên trong vỏ có sẵn**. KHÔNG tự vẽ nền/viền/thanh kéo —
làm vậy sẽ lệch chuẩn so với widget hệ thống.

### 4.1 Vỏ widget có sẵn (đừng đụng vào)
- `.wos-widget` — khung ngoài: kính mờ `backdrop-blur(28px)`, bo góc 18px, đổ bóng.
- `.wos-widget-bar` — thanh tiêu đề 11px/600 (đã hiện icon + title hộ bạn).
- `.wos-widget-body` — vùng nội dung: `flex:1; min-height:0; overflow:hidden`.
- `.wos-widget-resize` — góc resize dưới-phải.

**Component của bạn chỉ cần render 1 phần tử gốc:**

```js
function QuoteWidget() {
  return WOS.h('div', { className: 'wos-widget-body flex flex-col p-3' },
    WOS.h('p', { className: 'text-sm text-muted-foreground' }, '…nội dung…'),
  )
}
```

### 4.2 Bảng màu — dùng token Tailwind ngữ nghĩa (TUYỆT ĐỐI không hard-code)
| Muốn | Dùng | Cấm |
|---|---|---|
| Chữ chính | `text-foreground` | `text-zinc-900`, `#111` |
| Chữ phụ | `text-muted-foreground` | `text-gray-500` |
| Nền khối con | `bg-muted/60` | `bg-gray-100` |
| Nút chính | `bg-primary text-primary-foreground` | `bg-blue-600` |
| Đường phân cách | `border-input`, `border-black/10` | `#ccc` |

Dark mode tự động đúng khi (và chỉ khi) dùng token — đừng viết `dark:` tay trừ khi cần.

### 4.3 Typography & khoảng cách (theo widget mặc định)
- Chữ thường: `text-[11.5px]` – `text-sm`, `font-medium`.
- Con số Highlight: `font-bold tabular-nums tracking-tight` (như đồng hồ).
- Nhãn nhỏ: `text-[9px]` – `text-[11px] font-semibold text-muted-foreground`.
- Padding nội dung: `p-3` (nhỏ) / `p-4` (lớn). Khoảng cách giữa khối: `gap-1` – `gap-3`.

### 4.4 Hành vi
- **KHÔNG** dùng `position: fixed/absolute` che kín desktop — widget đã được WidgetLayer định vị sẵn.
- Chiếm 100% không gian `.wos-widget-body` (dùng `flex flex-col`, `w-full h-full`).
- Load dữ liệu bằng `WOS.api('/api/...')` (tự mang session cookie) — có state
  `loading` (`Loader2 animate-spin`) và `error` (chữ `text-muted-foreground`, có nút thử lại).
- setInterval/poll phải dọn trong cleanup của `useEffect`.
- Kích thước đề xuất tham khảo: widget "nhỏ" 220–250 × 160–200; "vừa" 250–360 × 200–320;
  "timeline" 360+ × 420+.

### 4.5 Ví dụ widget HOÀN CHỈNH (plugin.json runtime, ES5, chuẩn style)

```js
// main.js — Quote widget minh hoạ mọi quy ước trên
function QuoteWidget() {
  var h = WOS.h, s = WOS.useState, e = WOS.useEffect
  var st = s({ quote: 'Đang tải…', err: '' })
  var setSt = st[1]
  e(function () {
    WOS.api('/api/modules/notes').then(function (d) {
      setSt({ quote: (d.notes && d.notes[0] && d.notes[0].title) || 'Chưa có ghi chú nào', err: '' })
    }).catch(function () { setSt({ quote: '', err: 'Không tải được' }) })
  }, [])
  return h('div', { className: 'wos-widget-body flex flex-col items-center justify-center gap-2 p-3' },
    st[0].err
      ? h('p', { className: 'text-sm text-muted-foreground' }, st[0].err)
      : h('p', { className: 'text-center text-[13px] font-medium leading-relaxed' }, '“' + st[0].quote + '”'),
    h('p', { className: 'text-[10px] font-semibold uppercase tracking-wide text-muted-foreground' }, 'Ghi chú hôm nay'),
  )
}

WOS.registerApp({ component: QuoteWidget, windowSize: { w: 360, h: 240 } })
WOS.registerWidget({
  type: 'quote', title: 'Ghi chú hôm nay', icon: '💬',
  defaultW: 240, defaultH: 190, minW: 200, minH: 160,
  component: QuoteWidget,
})
```

---

## 5. Cài đặt & kiểm thử

1. Đóng gói zip (plugin.json ở GỐC zip hoặc trong 1 thư mục con duy nhất).
2. Settings → **Plugins** → *Cài plugin từ .zip* → chọn file → Bật plugin.
3. Mở Settings → Appearance → **Desktop & Dock** → Widget Manager → widget của bạn
   nằm trong mục **Widget từ plugin** → *Thêm*.
   (Hoặc chuột phải Desktop → *Thêm widget*.)
4. Kiểm tra console (F12): cảnh báo `[WOS] registerWidget: thiếu def.type/component`
   nghĩa là tham số sai — xem lại §3.

### Sự cố thường gặp
| Triệu chứng | Nguyên nhân | Xử lý |
|---|---|---|
| `WOS.registerWidget is not a function` | Lõi WOS < 1.1.0 (hệ legacy chưa có API) | Nâng cấp lõi; tạm thời dùng hệ plugin.json |
| Widget không hiện sau khi cài | Plugin `kind:'pet'` không được Desktop chạy entry | Đặt `kind: "app"` trong plugin.json |
| Style lệch chuẩn | Hard-code màu/nền thay token | Làm lại theo §4.2 |
| Widget mất sau F5 | Đăng ký async sau khi script load xong | Gọi `WOS.registerWidget` đồng bộ trong entry |

---

## 6. Checklist trước khi phát hành

- [ ] `plugin.json` đủ trường, `slug` chưa trùng, `kind: "app"`.
- [ ] Widget render root `.wos-widget-body` + `flex flex-col`, không nền cứng.
- [ ] 100% màu qua token (`text-foreground`, `bg-muted/60`, `bg-primary`…).
- [ ] Dark mode bật/tắt kiểm tra visually không cháy màu.
- [ ] State loading + error + cleanup interval.
- [ ] `defaultW/H` theo §4.4, min không vượt default.
- [ ] Mở thử widget, kéo/resize/F5 — vị trí giữ nguyên.

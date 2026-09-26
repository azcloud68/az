# WOS Plugin Development Guide / Hướng dẫn phát triển plugin

Phiên bản: 2.0 · Workspace OS v5

## 1. Plugin là gì?

Plugin (app cài thêm) là **gói ứng dụng** chạy phía trình duyệt. Từ v5 có 2 định dạng:

- **.zip (khuyên dùng)** — pack nhiều file: `plugin.json` (manifest) + mã JS + CSS + ảnh assets. Plugin "nặng" nhiều file bắt buộc dùng kiểu này.
- **JSON 1 file (kiểu cũ)** — manifest + mã `entry` inline trong 1 file JSON, tiện plugin nhỏ.

Ngoài ra có **thư mục plugin riêng trên máy chủ** (`<STORAGE_ROOT>/plugins/`) — copy code trực tiếp lên đó rồi bấm "Quét lại" trong Settings → Plugins, hệ thống tự đăng ký.

Sau khi admin cài (Settings → Plugins → Cài plugin mới), app xuất hiện ngay:

- **Cửa sổ app riêng** (kéo/resize/maximize như app gốc)
- **Icon trên Desktop** (bảng lưới, kéo đổi ô được)
- **Launchpad / Dock** (tuỳ chọn `showDock`)
- **Notification Center / Timeline / Activity** — qua API `WOS.notify` / `WOS.emitEvent`

Kiểu plugin:

| kind | Mô tả |
|------|-------|
| `app` | App cửa sổ — mở từ Desktop/Launchpad/Dock |
| `pet` | Nhân vật chạy trên desktop (mascot) — tự chạy, không cần mở cửa sổ |

## 2. Định dạng plugin

### 2a. Kiểu .zip (nhiều file — khuyên dùng)

```
my-plugin.zip
├── plugin.json        ← manifest (bắt buộc)
├── main.js            ← mã JS chính — khai "entry": "main.js"
├── style.css          ← (tuỳ chọn) "styles": ["style.css"] → tự chèn
└── assets/
    └── hero.png       ← (tuỳ chọn) lấy qua WOS.assetUrl('assets/hero.png')
```

- Zip có thể bọc tất cả trong 1 thư mục con duy nhất (GitHub zip style) — hệ thống tự nhận.
- Chống zip-slip: tên entry chứa `../` / `/` tuyệt đối bị từ chối.
- Giới hạn: zip ≤ 100MB, ≤ 3000 file, giải nén ≤ 200MB, entry ≤ 500KB.
- Sau khi cài, toàn bộ pack nằm ở `<STORAGE_ROOT>/plugins/<slug>/` (gỡ plugin = gỡ luôn pack).
- Cài lại cùng slug (bản mới) = nâng cấp: giữ enabled/config người dùng.

`plugin.json`:

```json
{
  "slug": "my-plugin",
  "name": "Tên app",
  "description": "Mô tả ngắn",
  "icon": "🧩",
  "color": "#8b5cf6",
  "kind": "app",
  "version": "1.1.0",
  "author": "Tên bạn",
  "desktopIcon": true,
  "showDock": false,
  "windowSize": { "w": 520, "h": 560 },
  "entry": "main.js",
  "styles": ["style.css"]
}
```

### 2b. Kiểu JSON 1 file (plugin nhẹ)

```json
{
  "slug": "my-plugin",
  "name": "Tên app",
  "description": "Mô tả ngắn",
  "icon": "🧩",
  "color": "#8b5cf6",
  "kind": "app",
  "version": "1.0.0",
  "author": "Tên bạn",
  "desktopIcon": true,
  "showDock": false,
  "windowSize": { "w": 520, "h": 560 },
  "entry": "<mã JS — xem mục 4>"
}
```

| Trường | Bắt buộc | Ghi chú |
|--------|----------|---------|
| `slug` | ✔ | `a-z0-9-`, 2-40 ký tự, duy nhất (cũng là moduleId) |
| `name` | ✔ | Tên hiển thị |
| `icon` | ✖ | Emoji (vd `🧮`) hoặc tên lucide (vd `calculator`) |
| `color` | ✖ | Màu nền icon gradient |
| `kind` | ✖ | `app` (mặc định) hoặc `pet` |
| `entry` | ✔ | Mã JS, tối đa 200KB |
| `windowSize` | ✖ | Kích thước cửa sổ, 280-1400 x 240-1200 |

## 3. Biến WOS (Plugin Runtime)

Mã entry chạy trong "use strict" với biến WOS:

### React (KHÔNG dùng JSX)

- `WOS.h` (React.createElement) + `WOS.createElement`
- `WOS.Fragment`, `WOS.useState`, `WOS.useEffect`, `WOS.useRef`, `WOS.useMemo`, `WOS.useCallback`

Vì plugin không được compile JSX lúc runtime -> dùng h():

```js
var h = WOS.h
function Demo() {
  var st = WOS.useState(0)
  return h('div', { className: 'demo', onClick: function(){ st[1](st[0]+1) } },
    h('span', null, 'Đã bấm ' + st[0] + ' lần'))
}
```

### Đăng ký app / nhân vật

- `WOS.registerApp({ component, windowSize })` -> app mở được từ Desktop/Launchpad/Dock
- `WOS.registerPet({ component, config })` -> component chạy full màn hình, tự quản vị trí (rAF)

### Hệ thống

| API | Mô tả |
|-----|-------|
| `WOS.notify({title, content, type, action})` | Gửi thông báo vào Notification Center (persist DB) |
| `WOS.emitEvent(type, payload)` | Phát event lên server Event Bus (Timeline + Activity) + client bus. Type tự prefix `plugin.<slug>.` |
| `WOS.openApp(moduleId, {route, key})` | Mở app hệ thống (tasks, notes, files...) |
| `WOS.api(path, opts)` | fetch helper chuẩn WOS (tự kèm auth) |
| `WOS.bus` | Event bus client: on / onAny / emit |
| `WOS.toast` | sonner: success/error/info(title, {description}) |
| `WOS.css(cssText)` | Chèn CSS — prefix class theo slug để tránh xung đột |
| `WOS.assetUrl(file)` | URL tài nguyên trong pack (vd `WOS.assetUrl('assets/hero.png')`) — phục vụ từ `/api/core/plugins/asset` |
| `WOS.storage.get/set/remove(key, value)` | localStorage riêng prefix `wos_plugin_<slug>_` |
| `WOS.clearCache()` | Xoá toàn bộ cache localStorage của plugin |
| `WOS.plugin` | `{ slug, name, config, version }` của plugin |

### Config

Admin chỉnh config JSON của plugin qua API PATCH. Plugin đọc khi khởi động:

```js
var speed = (WOS.plugin.config && WOS.plugin.config.speed) || 130
```

## 4. Ví dụ đầy đủ (Hello World)

```json
{
  "slug": "hello-world",
  "name": "Hello World",
  "icon": "👋",
  "color": "#f97316",
  "kind": "app",
  "version": "1.0.0",
  "author": "Bạn",
  "windowSize": { "w": 380, "h": 300 },
  "entry": "var h = WOS.h, useState = WOS.useState; WOS.css('.hw{padding:24px;font-size:22px;text-align:center}.hw button{margin-top:12px}'); function Hello(){ var st = useState(0); return h('div',{className:'hw'}, h('div',null,'Xin chào WOS! Bấm ' + st[0] + ' lần'), h('button',{onClick:function(){ st[1](st[0]+1) }},'Bấm tôi'), h('button',{onClick:function(){ WOS.notify({title:'Hello!',content:'Từ plugin hello-world'}) }},'Gửi thông báo')) } WOS.registerApp({ component: Hello })"
}
```

## 5. Gợi ý viết plugin pet (nhân vật desktop)

- Component chạy requestAnimationFrame; dùng `transform: translate3d()` để mượt.
- Đọc vị trí icon desktop để tương tác: `document.querySelectorAll('.wos-desktop-file, .wos-desktop-icon')` + getBoundingClientRect().
- Đừng đè lên icon: khi chạm -> nhảy qua / nhảy lên ngồi trên icon / quay đầu (xem plugin "Mèo Anime" dựng sẵn).
- Giới hạn vùng: menubar 34px trên, dock 92px dưới.

## 6. Cài / gỡ / nâng cấp

- **Cài**: Settings -> Plugins -> "Cài plugin mới" (file .json hoặc dán JSON). Chỉ admin.
- **Bật/tắt**: switch trong danh sách — tắt = đóng cửa sổ + ẩn icon ngay.
- **Gỡ**: nút Gỡ (plugin tự viết). Plugin dựng sẵn chỉ tắt được, không gỡ.
- **Nâng cấp**: gỡ bản cũ rồi cài JSON mới cùng slug.

## 7. Bảo mật

- Plugin do admin cài trên server tự host — coi như mã tin cậy (như cài .exe trên Windows).
- API hệ thống có guard: WOS.notify / emitEvent yêu cầu đăng nhập + plugin đang bật.
- Event plugin luôn prefix `plugin.<slug>.` để phân biệt event hệ thống.

## 8. Checklist trước khi phát hành

- [ ] Slug + version + author khai báo đúng
- [ ] Mọi class CSS prefix theo slug
- [ ] Dọn listener trong useEffect return (chống rò rỉ bộ nhớ)
- [ ] Test: mở/đóng cửa sổ nhiều lần, bật/tắt plugin, F5 lại trang
- [ ] WOS.notify và WOS.emitEvent hoạt động (kiểm tra Notification + Timeline)

# Atun Translator

Ứng dụng Android (Capacitor) nghe/dịch trực tiếp: bắt audio hệ thống, nhận dạng
giọng nói offline (Vosk), dịch bằng ML Kit / Google Translate free, hiện phụ
đề đè lên video đang phát trong app, và xuất transcript.

> App ID: `com.atun.translator` · `webDir`: `www` (xem `capacitor.config.ts`)

## Cấu trúc thư mục

```
.
├── www/                        # Toàn bộ giao diện web (Capacitor webview)
│   ├── index.html              # Trang chính: STT/dịch trực tiếp, player video, phụ đề
│   └── src/services/           # Cầu nối JS <-> plugin native (bridge)
│       ├── audio/NativeCaptureBridge.js
│       ├── translation/MlkitTranslateBridge.js
│       ├── tts/TtsBridge.js
│       └── export/ExportBridge.js
├── native/                     # Plugin/Activity/Service Kotlin (package com.atun.translator)
│   ├── MainActivity.kt
│   ├── BrowserActivity.kt      # Trình duyệt mini trong app
│   ├── AudioCapturePlugin.kt / AudioCaptureService.kt   # Bắt audio hệ thống (foreground service)
│   ├── VoskEngine.kt / MultiLangVoskEngine.kt / SttEngine.kt
│   ├── VadDetector.kt          # Voice Activity Detection
│   ├── MlkitTranslatePlugin.kt
│   ├── TtsPlugin.kt
│   ├── ModelManagerPlugin.kt   # Tải/quản lý model STT theo ngôn ngữ (on-demand)
│   ├── ExportPlugin.kt         # Xuất transcript (.srt/.txt)
│   ├── OverlayView.kt / OverlayWindowPlugin.kt
│   ├── ScreenOrientationPlugin.kt
│   ├── BrowserLauncherPlugin.kt / BrowserStorage.kt / AdBlockList.kt
│   └── AppLog.kt
├── scripts/                    # Script Python patch file Gradle/Manifest sinh ra bởi `npx cap add android`
│   ├── patch-kotlin.py         # Bật Kotlin support trong build.gradle
│   ├── patch-vosk-gradle.py    # Thêm dependency Vosk + maven repo Alphacephei
│   ├── patch-mlkit-gradle.py   # Thêm dependency ML Kit Translate
│   ├── patch-manifest.py       # Thêm permission cần thiết vào AndroidManifest.xml
│   └── patch-release-build.py  # signingConfigs.release + minify/shrink + Proguard keep-rules
├── supabase-functions/         # Edge Functions (Supabase) dùng làm backend phụ
│   ├── youtube-extract/        # Lấy stream/info video YouTube
│   └── groq-relay/             # Relay gọi Groq API (giấu API key khỏi client)
├── tests/                      # Test Python độc lập, không cần build APK
│   ├── vad/                    # Port tay VadDetector.kt sang Python + pytest
│   ├── translate/              # Test hàm dịch free (Google Translate không cần key)
│   └── model_catalog/          # Test parser danh mục model STT
├── capacitor.config.ts
├── package.json
└── .github/workflows/android-build.yml   # CI build APK debug/release (xem file rời đính kèm)
```

## Yêu cầu môi trường

- Node.js 20+, npm
- JDK 17
- Android SDK (qua Android Studio hoặc `cmdline-tools`)
- Python 3.10+ (chỉ để chạy `scripts/*.py` lúc setup, và `tests/`)

## Build local (từ đầu)

```bash
npm install
npx cap add android          # sinh thư mục android/ (không commit sẵn trong repo này)

# Copy code Kotlin vào đúng package path mà npx cap add android vừa tạo.
# QUAN TRỌNG: npx cap add android tự sinh sẵn 1 MainActivity.java mặc định trong thư mục này -
# phải xoá trước, nếu không dexBuilderDebug sẽ fail vì com.atun.translator.MainActivity bị
# định nghĩa 2 lần (.java mặc định + .kt tùy biến của mình).
mkdir -p android/app/src/main/java/com/atun/translator
rm -f android/app/src/main/java/com/atun/translator/MainActivity.java
cp native/*.kt android/app/src/main/java/com/atun/translator/

# Patch Gradle/Manifest (chạy đúng thứ tự)
python3 scripts/patch-kotlin.py
python3 scripts/patch-vosk-gradle.py
python3 scripts/patch-mlkit-gradle.py
python3 scripts/patch-manifest.py

# Build debug (không cần keystore)
cd android && ./gradlew assembleDebug
```

APK debug nằm ở `android/app/build/outputs/apk/debug/app-debug.apk`.

### Build release (đã ký + minify)

```bash
python3 scripts/patch-release-build.py
cd android && ./gradlew assembleRelease
```

Cần các biến môi trường sau (CI đọc từ GitHub Secrets, xem
`.github/workflows/android-build.yml`):

| Biến môi trường | Ý nghĩa |
|---|---|
| `RELEASE_KEYSTORE_PATH` | Đường dẫn file `.jks`/`.keystore` đã giải mã |
| `RELEASE_KEYSTORE_PASSWORD` | Mật khẩu keystore |
| `RELEASE_KEY_ALIAS` | Alias key ký |
| `RELEASE_KEY_PASSWORD` | Mật khẩu key |

Nếu không set các biến này, `patch-release-build.py` sẽ tự dùng lại debug
keystore mặc định của Android Gradle Plugin (chỉ để tự cài lên máy, **không**
dùng để phát hành Play Store).

## Chạy test (không cần Android SDK/APK)

```bash
cd tests/vad && pip install pytest && pytest -v
cd ../translate && pip install pytest requests && pytest -v
cd ../model_catalog && pip install pytest && pytest -v
```

## Supabase Edge Functions

Deploy riêng bằng Supabase CLI, không nằm trong quy trình build Android:

```bash
supabase functions deploy youtube-extract
supabase functions deploy groq-relay
```

`groq-relay` cần biến môi trường `GROQ_API_KEY` được set trong Supabase
project settings (Edge Function secrets) — không hard-code trong code.

## Ghi chú sửa lỗi gần đây

- **Khung phụ đề trong `www/index.html` bị hẹp bất thường so với video**: đã sửa 2 lớp -
  (1) quy đổi `widthPercent` ra px qua `getBoundingClientRect()` của `#videoStage` thay vì set
  `width` bằng CSS `%` trực tiếp trên phần tử `position:absolute` (hàm `applyCaptionSettings()`);
  (2) thêm thẻ `<meta name="viewport">` còn thiếu ở `<head>` - thiếu thẻ này khiến WebView
  Android render trang theo viewport ảo desktop rồi tự scale lại, kết hợp tính năng "font
  boosting" của WebView có thể khiến khung phụ đề (khối text hẹp) hiện sai tỉ lệ so với khung
  video dù JS đã tính đúng.
- **Nút "Phóng to" không hoạt động**: nguyên nhân THẬT là ở tầng native, không phải JS -
  Capacitor mặc định không cài `WebChromeClient.onShowCustomView()/onHideCustomView()`, nên
  `Element.requestFullscreen()` gọi từ JS không có nơi hiển thị, âm thầm không làm gì. Đã thêm
  `installFullscreenVideoSupport()` trong `native/MainActivity.kt` (kế thừa
  `BridgeWebChromeClient` của Capacitor, không mất hành vi mặc định như chọn file/xin quyền).
- **Tắt CC gốc YouTube (`cc_load_policy=0`)**: tham số đã đúng theo tài liệu YouTube, nhưng có
  2 giới hạn ngoài tầm code phía app - YouTube không phải lúc nào cũng tôn trọng tuyệt đối tham
  số này trên embed, và nếu video có phụ đề "cứng" (burned-in, ghi thẳng vào từng khung hình
  lúc dựng video) thì không có API/tham số nào của YouTube tắt được vì đó là pixel của video,
  không phải track CC thật.

## Ghi chú đóng gói do giới hạn upload trên GitHub mobile

Vì repo không commit sẵn thư mục `android/` (được sinh ra bởi
`npx cap add android` lúc build, theo đúng convention Capacitor), toàn bộ
source ở đây khá nhỏ (dưới 1MB) nhưng vẫn được chia thành nhiều file `.zip`
nhỏ để dễ tải lên bằng ứng dụng GitHub trên điện thoại.

**Lưu ý quan trọng: GitHub KHÔNG tự giải nén file `.zip` khi bạn tải lên** -
nó chỉ lưu nguyên file `.zip` trong repo, không tách thành các file/thư mục
thật. Có 2 cách dùng:

1. **Cách đơn giản nhất (khuyến nghị nếu chỉ dùng điện thoại):** cứ commit
   thẳng cả 4 file `atun-translator-part*.zip` vào **gốc repo** (cùng cấp với
   nơi sẽ có `package.json`), kèm `README.md` và
   `.github/workflows/android-build.yml`. Bước đầu tiên trong workflow CI
   (`Extract project source (nếu còn ở dạng .zip)`) sẽ tự tìm và giải nén
   các file `atun-translator-part*.zip` này ra gốc repo trước khi
   `npm install`/build - không cần tự giải nén tay trên điện thoại.
2. Nếu có sẵn app quản lý file hỗ trợ giải nén trực tiếp vào đúng thư mục
   repo đã clone (hoặc dùng máy tính), giải nén cả 4 phần vào cùng 1 thư mục
   gốc (đè lên nhau, không phần nào trùng tên file) rồi commit source thật
   (không cần giữ lại file `.zip`) cũng được - workflow tự phát hiện
   `package.json` đã có sẵn và bỏ qua bước giải nén.

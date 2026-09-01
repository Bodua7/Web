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
│   ├── ScreenOrientationPlugin.kt
│   ├── CaptionRelay.kt         # Đồng bộ câu phụ đề đang dịch sang overlay của "Web mini"
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
dùng để phát hành Play Store). Trường hợp này, workflow CI sẽ tự in cảnh báo
(`::warning::`, hiện nổi bật trên trang Actions) và đặt tên artifact tải về là
`atun-translator-release-UNSIGNED-debugkey` thay vì `atun-translator-release`
bình thường - để không lầm tưởng đó là bản đã ký thật bằng key phát hành.

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

## Ngôn ngữ hỗ trợ

Danh sách lấy đúng theo `ModelManagerPlugin.CATALOG` (STT) và
`MlkitTranslatePlugin.LANG_MAP` (dịch offline) hiện tại trong `native/` - mọi
model đều tải on-demand, không bundle sẵn trong APK.

### STT (Vosk) - 29 ngôn ngữ, tất cả đều <200MB

| Mã | Ngôn ngữ | Dung lượng |
|---|---|---|
| `en` | English | 128MB |
| `en-in` | English (Ấn Độ) | 36MB |
| `vi` | Tiếng Việt | 32MB |
| `ja` | 日本語 | 48MB |
| `ko` | 한국어 | 82MB |
| `zh` | 中文 (Trung Quốc) | 42MB |
| `ru` | Русский (Nga) | 45MB |
| `fr` | Français (Pháp) | 41MB |
| `de` | Deutsch (Đức) | 45MB |
| `es` | Español (Tây Ban Nha) | 39MB |
| `pt` | Português (Bồ Đào Nha) | 31MB |
| `tr` | Türkçe (Thổ Nhĩ Kỳ) | 35MB |
| `it` | Italiano (Ý) | 48MB |
| `nl` | Nederlands (Hà Lan) | 39MB |
| `ca` | Català (Catalan) | 42MB |
| `fa` | فارسی (Ba Tư/Iran) | 53MB |
| `uk` | Українська (Ukraine) | 73MB |
| `kz` | Қазақша (Kazakhstan) | 58MB |
| `eo` | Esperanto | 42MB |
| `hi` | हिन्दी (Ấn Độ - Hindi) | 42MB |
| `cs` | Čeština (Séc) | 44MB |
| `pl` | Polski (Ba Lan) | 50MB |
| `uz` | O'zbekcha (Uzbekistan) | 49MB |
| `gu` | ગુજરાતી (Gujarat) | 100MB |
| `tg` | Тоҷикӣ (Tajikistan) | 50MB |
| `te` | తెలుగు (Telugu) | 58MB |
| `ky` | Кыргызча (Kyrgyzstan) | 49MB |
| `ka` | ქართული (Georgia) | 45MB |
| `br` | Brezhoneg (Breton) | 70MB |

> Đã bỏ hẳn Ả Rập, Thuỵ Điển, Filipino, Hy Lạp khỏi catalog vì Vosk không có
> bản "small" cùng giọng chuẩn nào dưới 200MB cho 4 ngôn ngữ này (xem
> `ModelManagerPlugin.kt`). `en` trỏ thẳng sang bản lgraph ~128MB (gộp từ 3
> biến thể "en"/"en-lgraph"/"en-large" cũ) thay vì bản "small" 40MB kém chính
> xác hơn trước đây.

### Dịch offline (ML Kit) - 25 mã ngôn ngữ app khớp thẳng

Trùng với STT ở trên trừ 5 mã: `kz`, `uz`, `tg`, `ky`, `br` (ML Kit Translate
không hỗ trợ Kazakh/Uzbek/Tajik/Kyrgyz/Breton on-device - các ngôn ngữ này
vẫn nghe/STT được nhưng dịch sẽ tự fallback qua mạng: Google Translate
free/`groq-relay`).

### Ngôn ngữ dân tộc thiểu số Việt Nam

**Không được hỗ trợ** ở cả STT lẫn dịch offline - Vosk và ML Kit đều không có
model cho Tày, Thái, Mường, H'Mông, Gia Rai, Ê Đê, Ba Na, Chăm, Nùng, Dao,
Khmer (Nam Bộ), Cơ Ho... hay bất kỳ ngôn ngữ dân tộc thiểu số nào khác của
Việt Nam. Đây là các ngôn ngữ ít tài nguyên (low-resource), chưa có corpus
âm thanh/song ngữ công khai đủ lớn để 2 bên train model.

## "Web mini" (trình duyệt trong app)

Mở qua nút "🌐 Mở web mini" ở màn hình chính (`native/BrowserActivity.kt`) - trình duyệt
Android thuần, tách biệt hoàn toàn khỏi WebView Capacitor của màn hình dịch, dùng để vào bất
kỳ trang nào (vd YouTube) phát audio cho tính năng "Bắt âm thanh hệ thống" của app chính bắt
được. Các tính năng hiện có:

- Nhiều tab (tối đa 8), tự lưu/khôi phục danh sách tab đang mở nếu Android kill Activity ở nền
- Bookmark + lịch sử duyệt (xoá được lịch sử riêng, không ảnh hưởng bookmark)
- Chặn quảng cáo theo domain (bật/tắt được trong menu "⋮", mặc định bật)
- Icon khoá 🔒/⚠️/🌐 báo trang https/http/khác ngay trước ô địa chỉ
- Đè câu phụ đề đang dịch từ màn hình chính lên trên (qua `CaptionRelay`), fullscreen video
  HTML5/YouTube-iframe. **Đồng bộ cả khung dịch, không chỉ câu chữ**: độ trong suốt, cỡ chữ
  (Nhỏ/Vừa/Lớn), và tỉ lệ Rộng/Cao đang cấu hình ở màn hình chính (tự chọn đúng bộ thường/toàn
  màn hình theo trạng thái fullscreen) - Web mini tự quy đổi lại % đó theo kích thước vùng hiển
  thị của riêng nó (`native/CaptionRelay.kt`, `BrowserLauncherPlugin.updateCaption`,
  `applyCaptionFrame()` trong `BrowserActivity.kt`).
- **Chế độ Ngày/Đêm** (menu "⋮" → "🌙 Chế độ đêm" / "☀️ Chế độ ngày"): đổi màu khung giao diện
  (top bar, tab strip, ô địa chỉ...) và bật `forceDark` cho nội dung trang web đang mở ở mọi
  tab. Chỉ áp dụng riêng cho Web mini, không ảnh hưởng màn hình dịch chính. Lựa chọn được lưu
  lại (`BrowserStorage`), mở web mini lần sau vẫn giữ đúng chế độ đã chọn. `forceDark` cần
  Android 10 (Q) trở lên - máy cũ hơn vẫn dùng bình thường, chỉ riêng nội dung trang web sẽ
  không tự tối được, khung app vẫn đổi màu bình thường.

## Kết nối lại 1 chạm khi mất phiên nghe (chế độ "Bắt audio hệ thống")

Khi phiên nghe SYSTEM bị hệ thống dừng giữa chừng (app bị tắt/giải phóng bộ nhớ - không tự
phục hồi được vì token `MediaProjection` chỉ dùng 1 lần, giới hạn bảo mật của Android), banner
cảnh báo hiện ra có sẵn nút **"🔄 Kết nối lại ngay"** ngay trong banner - bấm 1 lần sẽ gọi lại
đúng luồng bắt đầu nghe với NGUYÊN lựa chọn ngôn ngữ/nguồn audio/ngắt câu thông minh hiện tại
(không cần chỉnh lại gì). Android vẫn sẽ hỏi lại quyền chia sẻ màn hình (dialog hệ thống) sau đó
- đây là giới hạn OS không có cách nào bỏ qua được, nhưng phía app chỉ còn đúng 1 thao tác.

## Whisper Cloud (Groq) - STT dự phòng khi Vosk offline chưa tốt

Bật mục **"🌩️ Dùng Whisper Cloud (Groq) khi cần độ chính xác cao hơn Vosk offline"** (dưới
"Ngắt câu thông minh") để bỏ qua Vosk hoàn toàn cho ngôn ngữ đang chọn - hữu ích khi model Vosk
offline nhận diện kém, hoặc chưa muốn tải model về máy. Cách hoạt động:

- Native (`AudioCaptureService.startCloudSttReadLoop`) chỉ dùng `VadDetector` (thuần thuật
  toán, không phải model) để biết lúc nào người nói tạm dừng, cắt audio đã ghi thành từng đoạn
  WAV (PCM16 mono 16kHz), base64 rồi gửi sang JS qua sự kiện `cloudSttSegment` - **không cần
  tải model STT nào cho ngôn ngữ đó**.
- JS (`transcribeCloudSegment` trong `www/index.html`) tự upload từng đoạn lên
  `groq-relay?op=transcribe` (Whisper `large-v3-turbo` qua Groq, endpoint này đã có sẵn từ
  trước cho tính năng khác) rồi đẩy text nhận được vào đúng pipeline dịch/hiển thị dùng chung
  với câu "final" của Vosk (xếp hàng tuần tự để giữ đúng thứ tự câu).
- Đánh đổi: cần mạng, có độ trễ vài giây/câu (chờ upload + Whisper xử lý), và **không có phụ đề
  "đang nói" (partial) hiển thị tức thời** như Vosk - chỉ có câu đã chốt hiện ra sau khi Whisper
  trả về.
- Ngưỡng cắt câu riêng cho cloud (`CLOUD_SEGMENT_PAUSE_MS=600ms`, cắt cứng sau
  `CLOUD_SEGMENT_MAX_MS=12s` nếu nói liên tục không ngừng) - khác hẳn `SMART_PAUSE_MS` (30ms,
  dùng để phụ đề Vosk chốt câu tức thời), vì mỗi lần cắt ở đây tốn 1 request mạng thật.
- **Chưa làm** (ngoài phạm vi lần sửa này): mở rộng danh sách "Ngôn ngữ đang nói" theo đúng tập
  ngôn ngữ Whisper hỗ trợ (rộng hơn nhiều so với 29 ngôn ngữ Vosk hiện có) - dropdown hiện tại
  vẫn dùng chung danh sách cũ, chỉ riêng việc BẬT cloud STT thì bỏ qua được bước tải model.

## Ghi chú sửa lỗi gần đây

- **"Khoảng lặng đầu video" gây phụ đề trễ/dính chữ**: nguyên nhân là
  `VadDetector.noiseFloor` không có chặn trên - nếu đầu video có đoạn không
  phải giọng nói nhưng cũng không im lặng tuyệt đối mà TĂNG DẦN (nhạc dạo
  kiểu crescendo), ngưỡng vào tiếng nói bám theo `noiseFloor` leo lên rất cao
  chỉ sau vài giây, có thể vượt quá RMS giọng nói thật khi người nói bắt đầu
  -> VAD không bao giờ báo "có voice" lại được nữa cho cả phiên -> smartPause
  (`AudioCaptureService`) bị kẹt vĩnh viễn -> phụ đề phải chờ endpointer nội
  bộ chậm hơn của Vosk. Đã thêm `MAX_NOISE_FLOOR = 1500.0` (xem
  `native/VadDetector.kt`) + 2 test hồi quy tái hiện đúng kịch bản này trong
  `tests/vad/test_vad_detector.py`.
- **Ngắt câu thông minh (`SMART_PAUSE_MS`)**: hạ tiếp từ 50ms xuống 30ms theo
  yêu cầu - phụ đề chốt câu nhanh/nhạy hơn, đổi lại dễ vỡ câu vụn hơn nếu
  người nói ngừng lấy hơi hơi lâu giữa 2 âm tiết cùng 1 từ.
- **Tuỳ chỉnh khung phụ đề**: thêm preset cỡ nhanh (Gọn/Vừa/Rộng, riêng cho
  từng mode thường/toàn màn hình) và pinch-to-resize (chụm/mở 2 ngón ngay
  trên khung phụ đề để đổi Rộng/Cao trực tiếp, dùng chung Pointer Events với
  kéo-thả vị trí đã có) - xem `www/index.html`, khối `setupCaptionDragAndPinch()`.
  Auto font scaling theo độ dài câu/khung đã có sẵn từ trước
  (`fitCaptionFontSize()`), không phải làm mới.
- **Cảnh báo mất phiên nghe (lost session)**: khi phát hiện phiên SYSTEM bị
  hệ thống dừng giữa chừng, giờ tự cuộn tới + nhấp nháy viền nút "Bắt đầu
  nghe" thay vì chỉ hiện dòng cảnh báo tĩnh (dễ bị bỏ sót nếu nút nằm ngoài
  viewport lúc mở app). Lưu ý: chế độ SYSTEM không thể tự phục hồi hoàn toàn
  bằng code - Android bắt buộc xin lại quyền `MediaProjection` mỗi lần (giới
  hạn bảo mật OS, không phải thiếu sót), chế độ MIC thì tự phục hồi được qua
  `START_STICKY`.
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
- **Gọn lại catalog model STT** (xem `ModelManagerPlugin.kt`): gộp 3 biến thể tiếng Anh
  "en" (small, 40MB)/"en-lgraph" (128MB)/"en-large" (1.8GB) thành đúng 1 mã "en" duy nhất trỏ
  sang bản lgraph ~128MB (chính xác gần tương đương bản 1.8GB cũ nhưng nhẹ hơn nhiều, không
  còn giữ bản 40MB kém chính xác lẫn bản 1.8GB quá nặng để tải on-demand). Đồng thời bỏ hẳn Ả
  Rập/Thuỵ Điển/Filipino/Hy Lạp khỏi catalog vì cả 4 không có bản "small" dưới 200MB - catalog
  còn 29 ngôn ngữ, model lớn nhất chỉ ~128MB. `MlkitTranslatePlugin.kt` và UI (`index.html`)
  đã dọn theo cho khớp (xem mục "Ngôn ngữ hỗ trợ" phía trên).

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

   > **Lưu ý:** workflow CI hiện tại chỉ nhận diện file theo đúng mẫu tên
   > `atun-translator-part*.zip`. Nếu trước đây bạn từng dùng cách gộp
   > thành 1 file `atun-translator-source.zip` duy nhất, cách đó **không
   > còn được workflow tự nhận diện** - hãy xoá file gộp cũ đó khỏi gốc
   > repo nếu còn, để tránh nhầm lẫn (nó sẽ không được giải nén và bước
   > kiểm tra `package.json` sẽ báo lỗi).
2. Nếu có sẵn app quản lý file hỗ trợ giải nén trực tiếp vào đúng thư mục
   repo đã clone (hoặc dùng máy tính), giải nén cả 4 phần vào cùng 1 thư mục
   gốc (đè lên nhau, không phần nào trùng tên file) rồi commit source thật
   (không cần giữ lại file `.zip`) cũng được - workflow tự phát hiện
   `package.json` đã có sẵn và bỏ qua bước giải nén.

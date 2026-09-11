# Translator (com.atun.translator) — bản "Live API only"

App Android dịch phụ đề/giọng nói theo thời gian thực khi xem video hoặc nghe ai đó nói,
bằng cách bắt audio trên máy rồi gửi lên các dịch vụ Cloud (Whisper qua Groq để nhận diện
giọng nói, chuỗi Groq/Gemini/DeepL/Azure/Google Translate free để dịch). Xây bằng
**Capacitor 6 + Kotlin**, không dùng bundler JS (index.html/app.js/app.module.js thuần).

> Đây là bản đã **gỡ toàn bộ pipeline dịch/nhận diện chạy trên máy (on-device)** — không còn
> Vosk, ML Kit, TTS, xuất phụ đề (.srt/.txt/.vtt), quản lý model, overlay hệ thống, hay trang
> web-mini đi kèm. Chỉ còn đúng 1 đường: **Cloud STT (Whisper qua groq-relay) + chuỗi API dịch
> văn bản**. Nếu cần khôi phục các tính năng đã gỡ, xem lại lịch sử các bản zip cũ.

## Thay đổi gần đây (2026-09-10)

- **Hồi sinh "Dịch hội thoại 2 chiều" (`MODE_MIXED`)** — trước đây bị gỡ cùng đợt bỏ Vosk (chỉ
  chạy được với STT offline lúc đó), giờ chạy lại được trên Cloud STT: bắt ĐỒNG THỜI mic (mình
  nói) + audio hệ thống (đối phương), 2 luồng Cloud STT độc lập, dịch 2 CHIỀU NGƯỢC NHAU (system
  → tiếng Việt, mic → ngôn ngữ đối phương). Cần xin CẢ 2 quyền (RECORD_AUDIO trước, rồi
  MediaProjection) — xem `startMixedCapture()` trong `CaptureSourceManager.kt`.
- **Sửa lỗi đơn vị VAD hangover**: `HANGOVER_FRAMES` (đếm số khung, phụ thuộc kích thước buffer
  `AudioRecord` — khác nhau giữa các thiết bị) → `HANGOVER_MS=250` (đếm thời gian thật, nhất
  quán mọi máy) trong `VadDetector.kt`.
- **Audit + vá 2 vấn đề tự phát hiện khi đóng vai reviewer khắt khe**: (1) `mediaProjection`
  không được dọn nhất quán ở vài nhánh lỗi build audio hệ thống — không leak vĩnh viễn (vẫn được
  `onDestroy()` dọn muộn) nhưng không nhất quán, giờ `buildSystemAudioRecord()` tự dọn ở MỌI
  nhánh lỗi của chính nó; (2) khối dựng `AudioRecord` cho micro từng bị copy-paste giữa
  `startMicCapture`/`startMixedCapture` — tách thành `buildMicAudioRecord()` dùng chung.
- **Chưa build/test thật trên máy** — mọi thay đổi trên chỉ được kiểm tra tĩnh (đếm ngoặc
  Kotlin, `node --check` cho JS), CHƯA qua kotlinc/gradle thật. Đặc biệt rủi ro AEC (khử tiếng
  vọng) khi mic + system cùng hoạt động ở `MODE_MIXED` — hành vi thật trên thiết bị thật CHƯA
  được xác nhận.

## Kiến trúc

**Bắt audio** (3 chế độ, chọn 1 khi bắt đầu phiên):
- `MODE_SYSTEM` — bắt audio đang phát bởi app khác trên máy (YouTube, VLC...) qua
  `AudioPlaybackCaptureConfiguration` (Android 10+, cần quyền MediaProjection, xin lại mỗi lần).
- `MODE_MIC` — ghi âm qua micro máy (nghe qua loa ngoài), chỉ cần quyền `RECORD_AUDIO`
  (permission thường, xin 1 lần), tự khởi động lại được sau khi bị OS kill.
- `MODE_MIXED` — "Dịch hội thoại 2 chiều": bắt ĐỒNG THỜI `MODE_MIC` + `MODE_SYSTEM` (xem mục
  "Thay đổi gần đây" ở trên). Cần CẢ 2 quyền, KHÔNG tự khởi động lại được sau khi bị OS kill
  (giống `MODE_SYSTEM`, vì cũng cần token MediaProjection).

**Nhận diện giọng nói (STT):** duy nhất qua Cloud — cắt câu bằng VAD trên máy
(`VadDetector.kt`), đẩy từng đoạn audio lên Whisper qua Supabase Edge Function `groq-relay`
(`SttReadLoop.kt` phía native bắn sự kiện `cloudSttSegment`, JS tự lo upload).

**Dịch văn bản:** chuỗi fallback tuần tự trong `www/app.js` (`translateText()`), dừng lại ở
provider đầu tiên thành công:

1. Groq (LLM) — `op=translate`
2. Gemini (LLM) — `op=translateGemini`, dự phòng thứ 2 sau Groq
3. DeepL — `op=translateDeepl`
4. Azure Translator — `op=translateAzure`, **chỉ** dùng cho `kk`/`uz`/`ky`
   (xem `AZURE_TARGET_LANG_MAP`)
5. Google Translate free — chặng cuối cùng, không qua `groq-relay`

Groq và Google Translate free đều có circuit breaker riêng (mở sau vài lỗi liên tiếp, đóng
băng một lúc) để tránh spam quota khi provider đang lỗi. Kết quả dịch được cache lại
(localStorage) theo cặp (text, sourceLang, targetLang).

### Sơ đồ luồng dữ liệu

```
[App khác đang phát audio]  hoặc  [Micro máy]
              │ (MediaProjection)         │ (RECORD_AUDIO)
              ▼                           ▼
       CaptureSourceManager.kt  (dựng nguồn thu, chọn thiết bị)
                        │
                        ▼
              SttReadLoop.kt  (đọc AudioRecord liên tục)
                        │
                        ▼
              VadDetector.kt  (cắt câu: im lặng -> có tiếng nói -> im lặng)
                        │  (mỗi đoạn cắt xong, đóng gói WAV)
                        ▼
         sự kiện "cloudSttSegment"  (native -> JS, qua Capacitor bridge)
                        │
                        ▼
   www/app.js  upload đoạn WAV lên  groq-relay?op=transcribe  (Supabase Edge Function)
                        │  (Edge Function gọi Whisper qua Groq, trả lại text)
                        ▼
              www/app.js: translateText(text)
                        │
                        ▼
   Groq -> Gemini -> DeepL -> Azure(kk/uz/ky) -> Google Translate free (dừng ở bước đầu thành công)
                        │
                        ▼
              Hiển thị phụ đề lên màn hình (index.html)
```

### Permissions (AndroidManifest, patch bởi `patch-manifest.py`)

| Permission | Vì sao cần |
|---|---|
| `RECORD_AUDIO` | Bắt audio qua micro (`MODE_MIC`/`MODE_MIXED`) và bắt buộc phải có (dù không dùng `MODE_MIC`) để `AudioPlaybackCaptureConfiguration` hoạt động ở `MODE_SYSTEM`. |
| `FOREGROUND_SERVICE` + `FOREGROUND_SERVICE_MICROPHONE` | `AudioCaptureService` chạy nền dạng foreground service (bắt buộc từ Android 9+ để không bị hệ điều hành kill khi app xuống nền), loại `microphone` bắt buộc khai báo riêng từ Android 14. Khai báo sẵn `foregroundServiceType="mediaProjection\|microphone"` (kết hợp OR bit-mask) để phục vụ cả `MODE_MIXED` — xem `CaptureNotification.kt`. |
| `POST_NOTIFICATIONS` | Hiển thị notification bắt buộc của foreground service (Android 13+ notification là permission phải xin, không tự động cấp). |
| `INTERNET` | Gọi `groq-relay` (Supabase Edge Function) + Google Translate free. |

## Cấu trúc thư mục

```
native/                  File .kt copy đè vào module Android khi build (xem yml)
  MainActivity.kt           BridgeActivity, đăng ký 2 plugin còn lại + hỗ trợ fullscreen video
  AudioCapturePlugin.kt     Cầu nối JS <-> AudioCaptureService (xin quyền, start/stop capture)
  AudioCaptureService.kt    Foreground service điều phối bắt audio + đọc STT
  CaptureSourceManager.kt   Dựng nguồn thu (MediaProjection / micro / cả 2 cùng lúc ở
                             MODE_MIXED), chọn thiết bị mic ngoài. buildSystemAudioRecord()/
                             buildMicAudioRecord() dùng chung cho mọi mode cần đến từng loại.
  SttReadLoop.kt            Vòng lặp đọc AudioRecord, cắt đoạn, đẩy sang Cloud STT - dùng chung
                             cho cả luồng mic lẫn system ở MODE_MIXED (tham số hoá qua channel/
                             assignThread, xem CaptureSourceManager.startMixedCapture).
  SttSegmentCutter.kt        Logic THUẦN (không đụng AudioRecord/Thread) quyết định khi nào cắt
                             đoạn audio thành 1 segment gửi Cloud STT - tách ra để test JUnit
                             thuần được, xem native-test/SttSegmentCutterTest.kt.
  AudioPcmUtils.kt           Hàm thuần build WAV file/chuyển đổi PCM - xem
                             native-test/AudioPcmUtilsTest.kt.
  AudioInputDevices.kt      Liệt kê/mô tả thiết bị micro ngoài
  CaptureNotification.kt    Notification foreground - text/foregroundServiceType khác nhau theo
                             MODE_MIC/MODE_SYSTEM/MODE_MIXED
  VadDetector.kt            Voice Activity Detection để cắt câu (pure Kotlin, không phụ thuộc
                             Android framework — xem native-test/VadDetectorTest.kt). Hangover
                             tính theo THỜI GIAN THẬT (HANGOVER_MS), không đếm số khung.
  ScreenOrientationPlugin.kt Khoá/mở xoay màn hình khi fullscreen
  AppLog.kt                 Ghi log lỗi cục bộ trên máy

native-test/              Unit test JUnit, copy vào android/app/src/test/ lúc CI (KHÔNG đóng
                           gói vào APK).
  VadDetectorTest.kt         Test VadDetector (pure Kotlin, không cần Robolectric) - gồm cả test
                              hangover theo mốc thời gian tường minh (còn true ở 249ms, chuyển
                              false ở 250ms).
  SttSegmentCutterTest.kt    Test logic thuần quyết định khi nào cắt đoạn (SttSegmentCutter.kt)
  AudioPcmUtilsTest.kt       Test hàm thuần build WAV/chuyển đổi PCM (AudioPcmUtils.kt)
  AudioCaptureServiceTest.kt Test Robolectric cho AudioCaptureService - CHỈ 2 kịch bản an toàn
                              (onStartCommand với intent null/ACTION_STOP khi chưa từng start
                              capture) - xem comment đầu file về các nhánh CHƯA test (đụng
                              AudioRecord/MediaProjection thật, dễ ra "xanh giả" nếu viết ẩu).
  AudioInputDevicesTest.kt   Test Robolectric cho phần chọn thiết bị mic ngoài (dùng thật bởi
                              CaptureSourceManager.kt) - nhãn hiển thị, lọc micro trong máy,
                              thứ tự ưu tiên USB/dây/Bluetooth SCO.

www/                      Web assets (webDir của Capacitor), không qua bundler
  index.html
  app.js                    Script thường (không module)
  app.module.js             Script type="module"
  style.css
  src/services/audio/NativeCaptureBridge.js  Cầu nối JS <-> AudioCapturePlugin

.maestro/translate-manual.yaml  Flow Maestro test luồng "dịch thủ công" - xem mục Kiểm thử,
                                 CHƯA tự chạy thử được thật (không có emulator để verify)

eslint.config.js           Rule ESLint (lỏng, chỉ bắt lỗi rõ ràng) cho www/app.js + app.module.js

supabase-functions/groq-relay/
  index.ts                   Edge Function chính (Deno.serve) — proxy Groq/Gemini/DeepL/Azure,
                              giấu API key phía server, kiểm tra JWT qua verifyUser()
  pure.ts                    Hàm/hằng số THUẦN tách từ index.ts (không network, không
                              Deno.serve/Supabase client) để Deno.test import an toàn — xem
                              comment đầu file. Gồm: circuit breaker Groq, jsonOk/jsonError/
                              sanitizeClientError, và phần "gọi API + parse response" của cả 4
                              provider dịch — translateViaDeepl/translateViaAzure (nhận fetch
                              giả lập được, phần check/ghi quota DB vẫn ở index.ts) và
                              translateViaGemini/groqFetchWithFallback (tách TOÀN BỘ, không
                              đụng DB nên tách hết được, không chỉ phần gọi API).
  pure.test.ts                Deno test cho pure.ts (mock fetch cho cả 4 provider dịch)
  migration_*.sql             Migration Postgres theo dõi quota tháng/ngày cho DeepL/Azure

scripts/                  Script Python chạy trong CI để patch source Android trước khi build
  patch-kotlin.py            Copy .kt vào module + patch liên quan
  patch-manifest.py          Patch AndroidManifest.xml (quyền, foregroundServiceType...)
  patch-version.py            Ghi versionCode (= số run CI)/versionName (= package.json) thật
                               vào android/app/build.gradle
  patch-robolectric.py        Thêm dependency Robolectric + testOptions (test-only, không ảnh
                               hưởng APK thật) - luôn chạy, không gated bởi secret nào
  patch-firebase.py           Bật Crashlytics/Performance/Analytics - CHỈ chạy nếu có secret
                               GOOGLE_SERVICES_JSON_BASE64 (tuỳ chọn, xem mục Giám sát)
  patch-release-build.py     Patch cấu hình build release (ký, minify)

tests/                    Test Python (KHÔNG compile Kotlin/Deno thật — môi trường không có
                           kotlinc/Deno cài sẵn cục bộ; test dùng cách đọc trực tiếp source thật
                           (.ts/.kt) rồi so khớp bằng regex, để tự fail nếu source đổi mà test
                           không cập nhật theo, thay vì hand-port một bản dễ lỗi thời)
  translate/                 Test Google Translate free (fallback cuối)
  cloud_stt_segment/          Test logic cắt đoạn audio (segment_cutter, wav_builder)
  auth_token/                 Test parse header "Authorization: Bearer <token>" của groq-relay

capacitor.config.ts       appId com.atun.translator, webDir "www"
package.json              Không có bundler thật ("build" chỉ echo, index.html là static);
                           trường "version" là nguồn versionName duy nhất trong repo (xem
                           patch-version.py)
RELEASE_CHECKLIST.md      Checklist các bước trước khi build release (gửi kèm zip source)
android-build.yml         CI build APK (xem mục Build bên dưới; gửi RIÊNG kèm README,
                           KHÔNG nằm trong zip source)
maestro-ui-test.yml       Workflow tuỳ chọn (chỉ chạy tay) build APK debug + chạy Maestro flow
                           trên emulator - gửi RIÊNG, đặt vào .github/workflows/
.github/dependabot.yml    Tự tạo PR cập nhật dependency npm + GitHub Actions hàng tuần (gửi
                           riêng cùng android-build.yml — cũng thuộc .github/, không nằm
                           trong zip source)
.github/workflows/codeql.yml   Quét bảo mật tĩnh JS/TS (gửi riêng, lý do tương tự)
```

## Build (qua GitHub Actions, không cần máy tính)

1. Giải nén zip source, commit toàn bộ vào gốc repo (giữ nguyên cấu trúc thư mục).
2. Đặt 4 file gửi riêng vào đúng chỗ: `android-build.yml`, `codeql.yml` và
   `maestro-ui-test.yml` vào `.github/workflows/`, `dependabot.yml` vào `.github/` (ngang hàng
   `workflows/`, không phải bên trong).
3. Push lên nhánh `main` → CI tự chạy 3 job nối tiếp trong `android-build.yml`:
   - **`validation`** (nhanh, không build Android): giải nén source (nếu còn ở dạng
     `atun-translator-part*.zip`, tự verify checksum nếu có commit kèm file `.sha256`) →
     `deno check` + `deno test` (groq-relay) → cài npm → ESLint (chỉ báo cáo) → `pytest` →
     Ruff (chỉ báo cáo) → Detekt (chỉ báo cáo) → gộp báo cáo lint thành 1 artifact riêng
     (`lint-reports-<run_number>`).
   - **`build-debug`**: scaffold Android + copy `native/*.kt` + `native-test/*.kt` + patch
     manifest/Robolectric/Firebase(nếu có secret)/version → `./gradlew testDebugUnitTest` (fail
     thì dừng, không build tiếp) → `./gradlew assembleDebug`. Chạy trên mọi push/PR, hoặc
     workflow_dispatch không tick `build_release`.
   - **`build-release`**: chỉ chạy khi workflow_dispatch có tick `build_release`, và chỉ từ
     nhánh `main`/`release/*`. Cũng patch Firebase (nếu có secret) + version, không chạy unit
     test (đã chạy ở `build-debug` trên cùng commit rồi). Có `environment: release` để bật phê
     duyệt thủ công (xem mục "Bật phê duyệt thủ công cho release" bên dưới — chưa tạo Environment
     thì job vẫn chạy bình thường, không bị chặn).
4. Muốn build APK **release** (đã ký + minify + verify chữ ký bằng `apksigner`, tạo kèm GitHub
   Release): xem `RELEASE_CHECKLIST.md`.
5. Tải APK ở 1 trong 2 chỗ: tab Actions → Artifacts (tên kèm version/commit/số lần chạy, kèm
   `build-info.json`; debug giữ 7 ngày, release giữ 30 ngày), hoặc tab **Releases** của repo
   (chỉ với bản release, không tự xoá, link tải công khai nếu repo public).
6. Muốn chạy thử UI test (Maestro, tuỳ chọn — xem mục Kiểm thử): tab Actions → chọn workflow
   "Maestro UI test (tuỳ chọn)" → Run workflow. Không tự chạy trên push/PR.

### Chạy test ở máy khác (không qua CI), nếu có máy cài sẵn công cụ

- Python: `pip install "pytest>=8.0,<9" requests && pytest tests/translate tests/cloud_stt_segment tests/auth_token -v`
- Deno (groq-relay): `deno test --allow-env=GROQ_API_KEY,GROQ_API_KEY_2 supabase-functions/groq-relay/pure.test.ts`
- Kotlin (`VadDetectorTest.kt`): cần scaffold Android trước — không chạy độc lập được, xem các
  bước "Add Android platform" → "Copy native Kotlin sources" → "Copy Kotlin unit tests" →
  `./gradlew testDebugUnitTest` trong `android-build.yml` để làm lại thủ công.

### Bật phê duyệt thủ công cho release

Settings → Environments → New environment → đặt tên đúng `release` → Required reviewers → thêm
tài khoản GitHub của bạn (hoặc người khác) → Save. Từ lần chạy `build_release` sau, job sẽ dừng
chờ tới khi có người bấm Approve trong tab Actions mới build tiếp — GitHub tự ghi log ai đã
duyệt, lúc nào.

### Branch protection (khuyến nghị, làm thủ công trong Settings, không có trong file cấu hình)

Settings → Branches → Add rule → nhánh `main` → bật "Require a pull request before merging" +
"Require status checks to pass before merging" (chọn job `validation` và `build-debug`) — CodeQL
và Dependabot không nằm trong repo dưới dạng file cấu hình được vì đây là setting phía GitHub,
không phải thứ gửi kèm zip/yml được.

## Kiểm thử

- **Python** (`tests/`): chạy thật trong CI (`pytest`), xem mô tả từng bộ ở "Cấu trúc thư mục".
- **Deno** (`supabase-functions/groq-relay/pure.test.ts`): test các hàm thuần (không network) —
  format ngày tháng, đọc IP client từ header, circuit breaker, dựng response JSON, rút gọn lỗi
  trước khi trả về client. KHÔNG test các hàm gọi thật Groq/Gemini/DeepL/Azure (`handleTranscribe`,
  `handleTranslate`...) — cần mock HTTP call, chưa làm ở đợt này.
- **Kotlin** (`native-test/`): `VadDetectorTest.kt` test đầy đủ logic cắt câu của VAD (ngưỡng
  vào/ra tiếng nói, hangover giữ đuôi câu, nhiễu nền không bị coi là giọng nói, reset).
  `AudioCaptureServiceTest.kt` dùng Robolectric — 2 kịch bản an toàn (`onStartCommand` với
  intent null hoặc `ACTION_STOP` khi CHƯA từng start capture). `AudioInputDevicesTest.kt` cũng
  Robolectric — test đầy đủ phần **chọn thiết bị micro ngoài** mà `CaptureSourceManager.kt`
  thực sự gọi (`selectMicInputDevice()` → `AudioInputDevices.findBestExternalInputDevice()`/
  `describeInputDevice()`): đúng nhãn tiếng Việt theo loại thiết bị, bỏ qua micro/loa trong máy,
  đúng thứ tự ưu tiên USB > dây có dây > Bluetooth SCO, trả `null` khi không có thiết bị ngoài
  nào — dùng `AudioDeviceInfoBuilder`/`ShadowAudioManager.addInputDevice()` có sẵn của
  Robolectric (không cần reflection thủ công). `SttSegmentCutterTest.kt`/`AudioPcmUtilsTest.kt`
  test logic THUẦN tách riêng khỏi `SttReadLoop.kt` (khi nào cắt đoạn / build WAV) — không cần
  Robolectric.
  **Giới hạn thật sự, không phải "chưa làm kịp"**: phần `AudioRecord`/`MediaProjection` THẬT
  trong `CaptureSourceManager.kt` (`buildSystemAudioRecord`/`buildMicAudioRecord`, dùng chung
  cho MODE_MIC/SYSTEM/MIXED) và vòng lặp `AudioRecord.read()` thật trong `SttReadLoop.kt` —
  không có cách tạo `AudioRecord` "giả" có ý nghĩa qua unit test JVM (kể cả Robolectric), việc
  "giả lập" phần này dễ ra test "xanh giả" (pass mà không phản ánh gì về việc máy thật có bắt
  được audio hay không) hơn là hữu ích. Đây là ranh giới hợp lý giữa unit test và test cần thiết
  bị thật, không phải việc còn thiếu.
  **`MODE_MIXED` (2026-09-10) hiện KHÔNG có test nào** — kể cả 2 kịch bản an toàn kiểu
  `AudioCaptureServiceTest.kt` cũng chưa viết cho `startMixedCapture()`. Chỉ được rà bằng đọc
  code tĩnh (đếm ngoặc + đối chiếu chữ ký hàm), CHƯA build/chạy thật.

- **Deno mock HTTP** (`pure.test.ts`): ngoài các hàm thuần không-network, đã có test mock `fetch`
  cho **cả 4 provider dịch**:
  - `translateViaDeepl()` — thành công, chọn đúng base URL theo key free/trả phí, ném
    `DeeplQuotaExceededError` đúng lúc HTTP 456, lỗi thường ở mã khác, response rỗng, gửi đúng
    `source_lang`/`context` tuỳ chọn.
  - `translateViaAzure()` — thành công, ném `AzureQuotaExceededError` khi HTTP 403 HOẶC body
    chứa chữ "quota" (heuristic, xem comment trong `pure.ts`), lỗi thường khi 401, gửi đúng
    query `from` tuỳ chọn.
  - `translateViaGemini()` — thành công ở model đầu, tách đúng dòng `PINYIN:`, tự chuyển model
    dự phòng khi 401/403/429, dừng ngay (không thử model khác) khi lỗi loại khác (vd 400), đưa
    context vào prompt đúng.
  - `groqFetchWithFallback()` — thành công ở key đầu, tự chuyển key thứ 2 khi 401, mở circuit
    breaker khi cả 2 key đều 401/429, dừng ngay khi lỗi khác (vd 500, không tính vào breaker),
    throw ngay không gọi fetch khi breaker đang mở, báo lỗi rõ khi thiếu key.
  Cần chạy với `--allow-env=GROQ_API_KEY,GROQ_API_KEY_2` (xem lệnh đầy đủ ở mục "Chạy test ở
  máy khác" bên dưới) vì vài test case tự set/xoá tạm 2 biến môi trường này.
- **UI/end-to-end** (`.maestro/translate-manual.yaml`, chạy qua workflow `maestro-ui-test.yml`):
  đã viết 1 flow Maestro test luồng "dịch thủ công" (nhập text -> bấm Dịch -> có kết quả), CHẠY
  ĐƯỢC qua workflow riêng — nhưng **CHƯA tự chạy thử được thật** (không có Android
  SDK/emulator/Maestro CLI trong môi trường soạn flow này để verify) và gọi THẬT tới backend
  (tốn quota Groq/DeepL thật mỗi lần chạy, không mock) — vì vậy workflow này CHỈ chạy tay
  (`workflow_dispatch`), không tự động trên push/PR/lịch. Lần đầu chạy thử, nếu fail ở bước tìm
  `srcText`/`translateBtn`, xem hướng dẫn sửa selector ở đầu file `.maestro/translate-manual.yaml`
  (khả năng do Maestro cần selector khác cho nội dung trong WebView so với native view).
- **Lint** (ESLint/Ruff/Detekt): chạy trong job `validation`, hiện đang **chỉ báo cáo, chưa chặn
  build** (`continue-on-error: true` trong `android-build.yml`) vì cả 3 công cụ mới bật lần đầu,
  khả năng cao có sẵn nhiều finding cũ không liên quan gì tới thay đổi thực tế. Xem báo cáo ở
  artifact `lint-reports-<run_number>` (riêng Detekt — ESLint/Ruff in thẳng ra log của bước
  tương ứng). Khi nào đã xem/dọn bớt finding hiện tại, đổi `continue-on-error: true` thành
  `false` ở bước tương ứng trong `android-build.yml` để lint thật sự chặn build lúc có lỗi mới.

## Bảo mật

- **Keystore ký release**: tạo bằng `keytool` (có JDK) — `keytool -genkey -v -keystore
  release.keystore -alias <tên_alias> -keyalg RSA -keysize 2048 -validity 10000`, sau đó mã hoá
  base64 để dán vào secret: `base64 -w0 release.keystore` → dán kết quả vào secret
  `ANDROID_KEYSTORE_BASE64`. 3 secret còn lại (`RELEASE_KEYSTORE_PASSWORD`,
  `RELEASE_KEY_ALIAS`, `RELEASE_KEY_PASSWORD`) là các giá trị bạn tự đặt lúc tạo keystore ở
  trên. Cả 4 set ở Settings → Secrets and variables → Actions.
- **API key backend** (Groq/Gemini/DeepL/Azure): chỉ lưu trên Supabase Secrets, KHÔNG BAO GIỜ
  xuất hiện trong source/artifact/log CI — client gọi qua `groq-relay`, Edge Function tự đọc
  secret từ môi trường Supabase.
- **Xác thực request tới `groq-relay`**: qua JWT (`verifyUser()` trong `index.ts`) — bất kỳ ai
  có JWT hợp lệ (kể cả không phải app thật, ví dụ tự lấy JWT rồi gọi thẳng bằng curl) đều gọi
  được endpoint; đây là lý do `sanitizeClientError()` cắt bớt chi tiết lỗi trước khi trả về
  client (tránh lộ thông tin nội bộ provider ra ngoài) — xem comment trong `pure.ts`.
- **Dependabot** (`.github/dependabot.yml`): tự tạo PR cập nhật `package.json` + các action
  trong workflow hàng tuần. Không bật cho Python/Gradle vì chưa có manifest thật để scan (xem
  comment trong file).
- **CodeQL** (`.github/workflows/codeql.yml`): quét tĩnh JavaScript/TypeScript (`www/*.js`,
  `supabase-functions/*.ts`) trên mỗi PR + hàng tuần. Chưa quét Kotlin (lý do kỹ thuật trong
  comment đầu file đó).
- **Checksum source zip** (tuỳ chọn): nếu commit trực tiếp file `atun-translator-part*.zip`
  chưa giải nén, tạo kèm file `.sha256` để CI tự đối chiếu trước khi build — trên điện thoại có
  thể dùng app "aShell You": `sha256sum ten-file.zip > ten-file.zip.sha256`, commit cả 2 file.
  Không tạo thì CI vẫn chạy bình thường, chỉ bỏ qua bước verify (xem `android-build.yml`).

## Vận hành

- **Provider dịch báo hết quota / lỗi liên tục**: `groq-relay` tự ghi nhận vào bảng Postgres
  `translate_quota_status` (theo tháng cho DeepL/Azure) và trả lỗi có mã dạng
  `<PROVIDER>_ERROR_<status>` (vd `DEEPL_ERROR_403`) — client tự chuyển sang provider kế tiếp
  trong chuỗi fallback (xem mục Kiến trúc), người dùng thường không thấy gián đoạn trừ khi TẤT
  CẢ provider cùng lỗi. Muốn xem provider nào đang bị chặn: query bảng
  `translate_quota_status` trực tiếp trên Supabase Dashboard (SQL Editor).
- **Xem log app**: `adb logcat` lọc theo tag (native ghi qua `AppLog.kt` — log cục bộ trên máy,
  không tự gửi lên đâu cả) hoặc filter theo package `com.atun.translator`.
- **Kiểm tra kết nối tới Supabase**: mở
  `https://<project-ref>.supabase.co/functions/v1/groq-relay?op=debug` trực tiếp trên trình
  duyệt — liệt kê model Groq thật đang khả dụng trên tài khoản, xác nhận Edge Function đã
  deploy + đọc được secret đúng.
- **Circuit breaker "kẹt" (nghi ngờ mở nhưng không tự đóng lại)**: breaker phía server
  (`isGroqCircuitOpen()` trong `pure.ts`) tự đóng sau 60 giây kể từ lần fail thứ 3 liên tiếp,
  KHÔNG cần can thiệp thủ công — nếu vẫn mở kéo dài hơn nhiều, khả năng cao Groq đang lỗi thật
  (không phải breaker bị "kẹt").

## Giám sát (Firebase Crashlytics/Performance/Analytics — tuỳ chọn)

**Chưa có project Firebase → làm theo 6 bước sau:**

1. Vào [console.firebase.google.com](https://console.firebase.google.com) → "Add project" → đặt
   tên tuỳ ý (vd "atun-translator") → Google Analytics bật/tắt tuỳ bạn (tắt cũng không ảnh hưởng
   Crashlytics/Performance) → Create project.
2. Trong project vừa tạo, bấm icon Android (</>) → "Add app" → **Android package name PHẢI gõ
   đúng** `com.atun.translator` (khớp `applicationId`) → bỏ qua ô "Debug signing certificate"
   (không cần cho Crashlytics/Performance cơ bản) → Register app.
3. Bấm "Download google-services.json" → **KHÔNG commit file này vào git** (xử lý như keystore
   — không tuyệt đối bí mật nhưng để đồng nhất cách quản lý, dùng chung cơ chế secret) → mã hoá
   base64: trên máy tính `base64 -w0 google-services.json`; trên điện thoại dùng app "aShell
   You" (đã dùng để tạo checksum/keystore) chạy lệnh tương tự.
4. Dán kết quả vào GitHub secret tên **`GOOGLE_SERVICES_JSON_BASE64`** (Settings → Secrets and
   variables → Actions → New repository secret).
5. Xong — KHÔNG cần sửa gì thêm. Từ lần build tiếp theo (debug lẫn release), CI tự phát hiện
   secret này, tự patch Gradle bật Crashlytics + Performance Monitoring + Analytics (xem
   `scripts/patch-firebase.py`). Chưa set secret thì build vẫn chạy bình thường, chỉ là không
   có 3 tính năng này (không chặn build — khác keystore, đây là secret KHÔNG bắt buộc).
6. Crash/performance data xuất hiện trên Firebase Console (mục Crashlytics/Performance của
   project) sau khi cài APK có tích hợp lên máy thật và dùng thử — thường mất vài phút tới khi
   hiện lần đầu (Crashlytics đợi lần MỞ APP TIẾP THEO sau khi crash mới thật sự gửi báo cáo lên,
   không gửi ngay lúc crash).

**Đã có project Firebase rồi** → chỉ cần làm lại bước 2-4 ở trên nếu chưa từng thêm app Android
vào project đó, rồi set secret như bước 4.

## Backend (Supabase)

`groq-relay` giấu toàn bộ API key (Groq/Gemini/DeepL/Azure) phía server qua Supabase
Secrets — client chỉ gọi `https://<project-ref>.supabase.co/functions/v1/groq-relay?op=...`.
Cần tự deploy Edge Function này lên Supabase (`supabase functions deploy groq-relay`) và tự
set các secret key tương ứng trên Dashboard — bước đó không nằm trong workflow CI ở trên,
làm thủ công.

## Cài đặt môi trường phát triển (chỉ cần nếu build/test trên máy thật thay vì qua CI)

Toàn bộ hướng dẫn ở README này giả định build/test qua GitHub Actions (không cần máy tính riêng)
— mục này chỉ để tham khảo nếu sau này có máy thật muốn tự chạy local:

| Công cụ | Version dùng trong CI | Dùng để |
|---|---|---|
| Node.js | 20 | `npm install`, `npx cap add android` |
| JDK | 17 (Temurin) | Build Gradle/Android |
| Android SDK | Có sẵn trên GitHub-hosted runner | Build APK, `apksigner`, `aapt2` |
| Python | 3.13 | Script `scripts/patch-*.py`, `pytest`, `ruff` |
| Deno | v2.x | `deno check`/`deno test` cho `groq-relay` |

## Việc còn phải tự làm (không tự động qua CI)

- Deploy `groq-relay` lên Supabase + set secret API key (Groq/Gemini/DeepL/Azure) trên
  Supabase Dashboard.
- Chạy các migration `.sql` trong `supabase-functions/groq-relay/` lên Postgres của project.
- Dán `SUPABASE_ANON_KEY` thật vào `www/app.js` nếu còn để placeholder.
- Tạo Environment `release` + branch protection cho `main` trong Settings (xem mục Build) nếu
  muốn dùng phê duyệt thủ công/chặn merge trực tiếp.
- Tạo project Firebase + set secret `GOOGLE_SERVICES_JSON_BASE64` nếu muốn Crashlytics/
  Performance/Analytics (xem mục Giám sát) — hoàn toàn tuỳ chọn, không có cũng build bình
  thường.
- Chạy thử `maestro-ui-test.yml` lần đầu (workflow_dispatch) để xác nhận flow
  `.maestro/translate-manual.yaml` chạy đúng trên emulator thật — CHƯA tự verify được (xem mục
  Kiểm thử) — nếu fail, sửa lại selector trong file flow theo hướng dẫn ghi ở đầu file đó.
- Robolectric cho `SttReadLoop`/`CaptureSourceManager`: đã xác nhận đây là **giới hạn kỹ thuật
  thật sự** (không phải việc còn thiếu) — xem mục Kiểm thử.
- **Build + test thật `MODE_MIXED` trên máy Android thật** (2026-09-10, ưu tiên cao) — mọi thay
  đổi liên quan chỉ mới được đọc tĩnh, CHƯA qua kotlinc/gradle thật. Đặc biệt cần xác nhận hành
  vi AEC (khử tiếng vọng) khi mic + system audio cùng hoạt động — xem mục "Thay đổi gần đây".

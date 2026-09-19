# Atun Translator

Ứng dụng Android (Capacitor WebView) nghe/dịch trực tiếp: bắt âm thanh (mic hoặc âm thanh hệ
thống), chuyển giọng nói thành văn bản, dịch sang ngôn ngữ đích và hiện phụ đề trực tiếp
(live caption overlay) đè lên màn hình. Package: `com.atun.translator`.

## Tính năng chính

- **Bắt âm thanh**: qua mic (kể cả mic ngoài) hoặc âm thanh hệ thống (MediaProjection), chạy nền
  bằng foreground service (`AudioCaptureService`).
- **Speech-to-text**: có 2 chế độ — offline (Vosk, chạy trên máy) và Cloud STT qua Whisper (Groq),
  tự cắt đoạn audio theo khoảng lặng (`SttSegmentCutter`, có nhận diện "giọng ca sĩ khi hát" để
  tránh gửi nhạc dài lên Whisper tốn quota vô ích) hoặc theo độ dài tối đa.
- **VAD (Voice Activity Detection)**: `VadDetector` lọc khoảng lặng trước khi đưa vào STT, có cơ
  chế thích nghi noise floor (giới hạn trần để không bị "trôi" vô hạn khi nền ồn tăng dần).
- **Dịch đa tầng (fallback chain)**: ML Kit (on-device) → Groq LLM → Gemini → DeepL → Azure
  (chỉ kz/uz/ky) → Google Translate free — càng xuống dưới càng là phương án dự phòng khi tầng
  trên lỗi/hết quota.
- **Đọc to bản dịch (TTS)**: tuỳ chọn bật/tắt, tắt tiếng nhanh (nhớ lại mức âm lượng cũ), chỉnh
  âm lượng (0-150%), chọn giọng đọc. Trong app native dùng plugin `TtsBoostPlugin`
  (`native/TtsBoostPlugin.kt`, TextToSpeech hệ thống + tự khuếch đại PCM qua AudioTrack khi
  >100% — xem mục "Tăng âm lượng dịch vượt 100%" bên dưới); ngoài app (dev/test trên trình duyệt)
  fallback về Web Speech API (`speechSynthesis`, kẹp cứng ở 100%). Chỉ đọc câu dịch MỚI trong
  luồng nghe trực tiếp, không đọc kết quả dịch văn bản/file thủ công.
- **Công cụ phụ**: dịch văn bản/file thủ công, dán link video để phát kèm phụ đề trong app (nhúng
  qua YouTube IFrame Player API phía client, không cần backend riêng — có chỉnh âm lượng video
  gốc riêng, tách biệt với âm lượng giọng đọc bản dịch).
- Backend nhẹ chạy trên Supabase Edge Function (`groq-relay`) — làm proxy gọi Groq/Gemini/DeepL/
  Azure, quản lý rate-limit/quota theo IP lẫn theo user (JWT ẩn danh) và circuit breaker cho Groq.

## Cấu trúc repo

Repo **không commit thư mục `android/`** — thư mục này do `npx cap add android` tự sinh ra lúc
CI chạy. Toàn bộ source được nén thành 4 file zip ở gốc repo (để tải lên bằng điện thoại cho
gọn), CI tự giải nén trước khi build:

| File zip | Nội dung |
|---|---|
| `atun-translator-part1-native.zip` | `native/` (Kotlin: plugin, service, VAD, STT...) + `native-test/` (JUnit/Robolectric unit test) |
| `atun-translator-part2-www.zip` | `www/` (WebView: `index.html`, `app.js`, `app.module.js`, `style.css`) |
| `atun-translator-part3-config.zip` | `package.json`, `capacitor.config.ts`, `eslint.config.js`, `.maestro/translate-manual.yaml`, `scripts/` (các script patch chạy trong CI), `RELEASE_CHECKLIST.md` |
| `atun-translator-part4-backend.zip` | `supabase-functions/groq-relay/` (Edge Function + migration SQL) và `tests/` (Python test harness) |

Mỗi file zip đi kèm 1 file `.sha256` cùng tên — CI tự đối chiếu checksum trước khi giải nén.

Thư mục con quan trọng:

```
native/                     Kotlin: AudioCaptureService, VadDetector, SttSegmentCutter,
                             AudioPcmUtils, AudioInputDevices, plugin Capacitor...
native-test/                Unit test JUnit thuần + Robolectric cho native/
www/                         Giao diện WebView (không qua bundler, script thường)
scripts/                     Script Python CI dùng để patch Gradle/Manifest/version/Robolectric/
                             Firebase/release-build sau khi `npx cap add android`
eslint.config.js             Config ESLint 9 (flat config) - lint www/app.js + app.module.js,
                             tự khai tay global cần thiết (không phụ thuộc npm install)
.maestro/translate-manual.yaml  Flow Maestro cho maestro-ui-test.yml (test đường dịch văn bản
                             thủ công qua UI thật)
supabase-functions/groq-relay/  Backend relay dịch (Deno/TypeScript)
tests/                       Test harness Python (port tay từ Kotlin/JS sang Python để test)
.github/workflows/           android-build.yml, codeql.yml, maestro-ui-test.yml
.github/dependabot.yml
```

### `www/app.js` vs `www/app.module.js` — 2 scope KHÔNG chung nhau

`index.html` nạp `app.js` như script thường (`<script src="app.js">`) rồi `app.module.js` như ES
module (`<script type="module" src="app.module.js">`) ngay sau đó. Đây là **2 scope JS hoàn toàn
tách biệt** — biến/hàm `const`/`let`/`function` khai ở file này KHÔNG tự thấy được ở file kia, kể
cả khi trùng tên (vd. mỗi file tự `document.getElementById(...)` DOM ref riêng của nó, không dùng
chung 1 biến). Mọi thứ `app.module.js` cần gọi lại logic chỉ có ở `app.js` (đường dịch đa tầng
`translateText`/hàng đợi `queueTranslate`, lưu cài đặt `saveSettings`, gọi Whisper Cloud
`transcribeCloudSegment`) đều đi qua cầu nối `window.__xxx = xxx` gán ở cuối định nghĩa hàm trong
`app.js`, rồi `app.module.js` gọi qua `window.__xxx(...)` — KHÔNG viết lại 1 bản riêng trong
`app.module.js` để tránh 2 bản logic (đặc biệt thứ tự fallback dịch, schema `localStorage`) lệch
nhau dần khi chỉ sửa 1 bên. Quên bắc cầu (khai biến/hàm trực tiếp mà không qua `window.__`) từng
gây ra hàng loạt lỗi `ReferenceError`/ESLint `no-undef` thật ở runtime — xem mục Kiểm thử #4.

### Bố cục `index.html` — mọi công cụ phụ giờ nằm trong `#mainControlSection`

Theo yêu cầu UI/UX gộp gọn màn hình chính: khối dịch văn bản/file thủ công (`#srcText`,
`#targetLang`, `#translateBtn`, `#srcFile`...) đã dời vào **cuối `#mainSettingsBody`** — thu/mở
CHUNG 1 nút ▾/▸ với phần cài đặt nghe (`#mainSettingsHeader`). Khối dán link video
(`#videoUrlInput`) + "Tuỳ chỉnh khung phụ đề" (`#captionSettingsWrap`) dời xuống ngay dưới card
"Cài đặt nghe/dịch", trước nút `#mainToggleBtn` ("Bắt đầu nghe"). Cả 3 khối này giờ đứng chung
trong `#mainControlSection` (`style="display:none"` cho tới khi xong onboarding — xem
`mainControlSectionEl.style.display` trong `app.js`).

**Đánh đổi cần nhớ khi sửa tiếp phần này**: trước đây dịch văn bản/file và dán-link-video đứng
độc lập ở đầu trang, dùng được ngay cả khi CHƯA xong onboarding (chưa cấp quyền thông báo/pin) —
giờ cả 2 chỉ dùng được SAU khi xong onboarding, vì nằm trong `#mainControlSection`. Đổi có chủ
đích (theo đúng bố cục UI yêu cầu), không phải bug. **Đã xác nhận hoạt động đúng trên máy thật**
(dán link, phát video, Dừng video, mở "Tuỳ chỉnh khung phụ đề" đều bình thường sau khi gộp).

### Tăng âm lượng dịch vượt 100% — `TtsBoostPlugin`

Web Speech API (`speechSynthesis.volume`) chỉ nhận 0.0-1.0, trình duyệt/WebView tự kẹp về 1.0 nếu
đặt cao hơn — không có `speechSynthesis.captureStream()` hay API nào để JS lấy audio thật ra tự
khuếch đại. `android.speech.tts.TextToSpeech.speak()` + `Bundle` `KEY_PARAM_VOLUME` cũng bị kẹp
y hệt ở 1.0f (giới hạn của chính framework Android, không phải do web).

`native/TtsBoostPlugin.kt` né giới hạn đó bằng cách: khi âm lượng yêu cầu ≤100%, gọi thẳng
`tts.speak()` như bình thường (nhanh, gần như tức thời); khi >100%, tổng hợp giọng đọc ra file
WAV qua `TextToSpeech.synthesizeToFile()`, tự nhân biên độ từng mẫu PCM16 lên bằng
`applyGainToPcm16()` (`native/TtsGainUtils.kt`, có clip cứng ở `[-32768, 32767]` tránh tràn
số/wrap-around), rồi tự phát bằng `AudioTrack` dựng tay — né hẳn đường phát mặc định của TTS
framework. Đây là kỹ thuật các app "Volume Booster" trên Android hay dùng, cùng lý do: âm lượng
media/TTS hệ thống bị khoá cứng ở 100%. Trần khuếch đại đặt ở 150% (`TtsBoostPlugin.MAX_VOLUME`)
để giữ mức vỡ tiếng (distortion) ở mức chấp nhận được — khuếch đại vượt biên độ gốc của mẫu CHẮC
CHẮN gây vỡ tiếng ít nhiều ở mức nào đó, đây là đánh đổi vật lý không tránh được, không phải bug.

`www/app.js` tự chọn đường: dùng `TtsBoostPlugin` khi `Capacitor.isNativePlatform()` (trong app
thật), fallback về Web Speech API khi chạy ngoài app (dev/test trên trình duyệt máy tính — ở đó
vẫn kẹp cứng 100%, chấp nhận được vì dev/test không cần khuếch đại).

Lưu ý lịch sử: từng có 1 plugin tên `TtsPlugin` bị gỡ cùng đợt bỏ STT/dịch offline (xem comment
trong `MainActivity.kt`). `TtsBoostPlugin` là plugin MỚI viết lại từ đầu, mục đích khác hẳn
(khuếch đại được >100%, bản cũ không có khả năng này).

**Chưa test được trên máy Android thật** (không có Android SDK/emulator trong môi trường viết
code) — cần xác nhận: `getVoices()`/`speak()` gọi qua Capacitor bridge đúng tham số
(`call.getDouble`/`getString` đúng tên như JS truyền); `synthesizeToFile()` → đọc WAV → khuếch
đại → `AudioTrack` phát ra tiếng thật, không câm hay lỗi; mức vỡ tiếng ở 150% có chấp nhận
được không (nếu vỡ quá nặng, có thể cần hạ `MAX_VOLUME` thêm hoặc thêm soft-limiter thay vì
hard-clip).

### Âm lượng video gốc — khôi phục lại YouTube IFrame Player API

Trước đây phát video bằng cách set thẳng `iframe.src = 'https://www.youtube.com/embed/' +
videoId + ...'` — đơn giản nhưng trang cha KHÔNG có cách nào chỉnh âm lượng của iframe nhúng qua
đường đó. Theo yêu cầu "thêm chỉnh âm lượng gốc", đã khôi phục lại YouTube IFrame Player API
(`<script src="https://www.youtube.com/iframe_api">` trong `index.html`, `YT.Player(...)` trong
`app.js`) — API này có `player.setVolume(0-100)` thật.

**Lưu ý cho ai sửa tiếp phần video**: API này đã từng bị bỏ 1 lần (xem comment cũ, lúc đó chỉ
dùng để đọc `getCurrentTime()` phục vụ đồng bộ lyrics — tính năng đã gỡ, nên bỏ luôn API cho gọn).
Giờ khôi phục lại vì lý do KHÁC hẳn (chỉnh âm lượng). Tác dụng phụ cần nhớ:
`new YT.Player('videoPlayerFrame', {...})` **THAY HẲN** phần tử `<iframe id="videoPlayerFrame">`
gốc bằng iframe riêng của chính nó — mọi chỗ code cũ dựa vào giữ tham chiếu/id/style inline của
iframe gốc đều phải sửa theo (đã sửa 2 chỗ: listener `fullscreenchange` chuyển sang so sánh quan
hệ cha-con với `videoStageEl` thay vì so tham chiếu iframe cũ; CSS style/fullscreen-sizing của
iframe chuyển từ selector `#videoPlayerFrame` sang `#videoStage iframe`).

`ytPlayer` được giữ lại (không `destroy()`) sau khi bấm "Dừng video" — lần "Phát" sau chỉ
`loadVideoById()` thay vì tạo player mới, đỡ phải tải lại toàn bộ iframe YouTube từ đầu.

**Bug thật đã gặp trên máy (đã sửa)**: `Uncaught TypeError: ytPlayer.loadVideoById is not a
function`. Nguyên nhân: `new YT.Player(...)` trả về object NGAY LẬP TỨC nhưng các hàm
`loadVideoById()`/`setVolume()`/`stopVideo()` CHƯA TỒN TẠI trên object đó cho tới khi sự kiện
`onReady` bắn ra (bắt tay `postMessage` với iframe con xong — có thể mất 0.5-2s tuỳ mạng/máy).
Code cũ coi `ytPlayer` khác `null` là "dùng được ngay", sai — đã thêm cờ `ytPlayerReady`, mọi lệnh
gọi (`loadVideoById`/`setVolume`/`stopVideo`) đều kiểm tra cờ này trước, video bấm "Phát" trong
lúc đang chờ được xếp vào `pendingVideoIdForPlayer` thay vì gọi thẳng. Đã thêm luôn `onError` (mã
2/5/100/101/150 theo tài liệu YouTube) — trước đó video lỗi (ID sai/chặn nhúng) sẽ im lặng mãi
mãi vì `onReady` không bao giờ bắn, không có gì báo cho người dùng biết tại sao khung video cứ
đen im lìm.

**Còn lại chưa test được trên máy Android thật**: phóng to toàn màn hình; logic tự chuyển hướng
fullscreen (bấm nút fullscreen CỦA CHÍNH YouTube, không phải nút "Phóng to" của app) có còn nhận
diện đúng sau khi đổi cách kiểm tra hay không.

## Build & chạy local

Dự án chưa có bundler (xem `package.json`, script `build` chỉ echo) — `www/` là HTML/JS thường,
không cần build bước riêng. Để dựng bản Android:

```bash
npm install
npx cap add android          # sinh thư mục android/ (không commit)
cp native/*.kt android/app/src/main/java/com/atun/translator/
python3 scripts/patch-kotlin.py
python3 scripts/patch-manifest.py
cd android && ./gradlew assembleDebug
```

Trong thực tế các bước này do CI làm tự động (xem phần dưới) — chỉ cần chạy tay khi muốn debug
trực tiếp trên máy có Android Studio/SDK.

## Kiểm thử (đã chạy trong CI)

### 1. Unit test Kotlin (JVM, `./gradlew testDebugUnitTest`)

Chạy ở cả job `build-debug` và `build-release` (release build luôn bị chặn nếu test đỏ).

- **JUnit thuần** (không cần Robolectric, vì không đụng API `android.*`):
  - `VadDetectorTest.kt` — gồm cả test cơ chế trần `MAX_NOISE_FLOOR` (nền tăng dần liên tục
    không được để noise floor chạy vô hạn).
  - `SttSegmentCutterTest.kt` — 2 điều kiện cắt đoạn (đã có giọng + vừa im lặng đủ lâu, hoặc
    đoạn quá dài dù chưa im lặng) và phương án "nhận giọng ca sĩ khi hát". (Fix CI: 3 test của
    phương án "nhận giọng ca sĩ" từng tự fail vì chính test giả lập audio liên tục SAI cách — chỉ
    gọi `onFrame()` 1 lần rồi nhảy thẳng `shouldFlush(t + maxSegmentMs)`, khiến điều kiện "vừa
    pause" (được kiểm tra TRƯỚC điều kiện chạm trần thời lượng trong `shouldFlush()`) hiểu nhầm
    thành 1 pause thật. Đã sửa test dùng helper `simulateContinuousVoiceThenFlush()` bắn khung
    giọng nói mỗi 100ms suốt cả đoạn, đúng như comment gốc của test đã *nói* sẽ làm. Không đụng gì
    tới `SttSegmentCutter.kt` — logic gốc vẫn đúng, chỉ có test sai.)
  - `AudioPcmUtilsTest.kt` — encode PCM16 little-endian, đúng thứ tự byte.
  - `TtsGainUtilsTest.kt` — nhân biên độ mẫu PCM16 theo hệ số khuếch đại (`applyGainToPcm16()`,
    dùng cho `TtsBoostPlugin` khi âm lượng dịch >100%), gồm cả case tràn số dương/âm phải clip
    đúng về `Short.MAX_VALUE`/`MIN_VALUE` chứ không wrap-around.
- **Robolectric** (giả lập khung Android trên JVM):
  - `AudioInputDevicesTest.kt` — logic chọn thiết bị input (mic ngoài vs mic máy).
  - `AudioCaptureServiceTest.kt` — các kịch bản `onStartCommand` an toàn (không đụng
    `AudioRecord`/`MediaProjection` thật).

### 2. Test Deno cho backend (`groq-relay`)

```bash
deno check supabase-functions/groq-relay/index.ts
deno test --allow-env=GROQ_API_KEY,GROQ_API_KEY_2 supabase-functions/groq-relay/pure.test.ts
```

Test các hàm thuần tách sang `pure.ts` (CORS headers, circuit breaker cho Groq, sanitize lỗi,
lấy IP client từ `X-Forwarded-For`, gọi DeepL/Azure/Gemini có giả lập `fetch`...) — **không**
import `index.ts` vì file đó tự mở `Deno.serve()` và khởi tạo Supabase client thật. Đã bổ sung
5 case ở audit đợt 5 cho 2 hàm vừa vá: `getClientIp()` với header toàn khoảng trắng (kể cả khi
IP thật ở phần tử trước, khoảng trắng ở phần tử cuối), và `sanitizeClientError()` với whitelist
`SERVER_MISSING_*` + message lạ bất kỳ phải trả `INTERNAL_ERROR`.

### 3. Test Python (`pytest tests/translate tests/cloud_stt_segment tests/auth_token -v`)

Cả 3 bộ đều là **bản dịch tay (port thủ công)** từ Kotlin/JS sang Python, để test được hành vi
logic thuần mà không cần máy Android/mic thật hay mạng thật — bắt buộc giữ đồng bộ tay với source
gốc mỗi khi sửa logic tương ứng:

- `tests/translate/` — retry/rate-limit của `translateGoogleFree()` (`www/app.js`) khi gặp HTTP
  429, đúng công thức backoff exponential (2s/4s/8s).
- `tests/cloud_stt_segment/` — đóng gói WAV (PCM16 → header WAV 44 byte, `AudioPcmUtils.kt`) và
  state machine cắt đoạn (`SttSegmentCutter.kt` + `SttReadLoop.kt`) — 27 test case, phủ cả thứ tự
  ưu tiên giữa 2 điều kiện cắt đoạn, `flushSegment()` cuối vòng lặp, và phương án "nhận giọng ca
  sĩ khi hát".
- `tests/auth_token/` — tách token khỏi header `Authorization: Bearer <token>` dùng trong
  `verifyUser()` — chỗ dễ viết sai nhất (thiếu dấu cách, quên `.trim()`, sai độ dài slice).

### 4. Static analysis (chỉ báo cáo, chưa chặn build)

`continue-on-error: true` cho cả 3 — kết quả upload thành artifact `lint-reports-<run>`:

- **ESLint** (`www/app.js`, `app.module.js`) — dùng `eslint.config.js` ở gốc repo (flat config,
  ESLint 9, tự khai tay danh sách global cần thiết vì bước lint không chạy `npm install`). Đã sửa
  8 lỗi `no-undef` thật (không phải false-positive) do `app.module.js` gọi thẳng
  `preferExternalMicEl`/`externalMicStatusEl`/`externalMicRowEl`/`mixedModeHintEl` (thiếu khai
  `getElementById` riêng) và `translateText`/`queueTranslate`/`saveSettings`/
  `transcribeCloudSegment` (chỉ định nghĩa bên `app.js`, khác scope) mà không qua cầu nối
  `window.__` — xem mục "app.js vs app.module.js" ở trên.
- **Ruff** (`scripts/`, `tests/`)
- **Detekt** (`native/*.kt`, tải jar từ Maven Central có verify checksum). Đã dọn phần lỗi
  thật/false-positive (giữ nguyên `MagicNumber`/`ReturnCount` — cần tự quyết ngưỡng hợp lý cho
  từng chỗ, không tự ý đổi):
  - `SwallowedException` (`AudioCaptureService.kt`, dừng `AudioRecord`) — thêm `Log.w(TAG, ..., e)`
    thay vì bắt rồi bỏ qua hoàn toàn.
  - `InvalidPackageDeclaration` (12 file `native/*.kt`) — false-positive: Detekt chạy trên
    `native/` phẳng TRƯỚC bước copy vào `com/atun/translator/` thật của `build-debug` — đã thêm
    `@file:Suppress("InvalidPackageDeclaration")` kèm giải thích ở từng file.
  - `UnusedPrivateMember` (4 hàm `handleXxxResult` trong `AudioCapturePlugin.kt`) —
    false-positive: đều gắn `@ActivityCallback`/`@PermissionCallback`, được Capacitor gọi qua
    reflection chứ không gọi trực tiếp trong code — đã thêm
    `@Suppress("UnusedPrivateMember")` kèm giải thích.

### 5. Maestro UI test (thủ công, `workflow_dispatch` riêng — `maestro-ui-test.yml`)

Build APK debug, cài lên emulator Android (Pixel 6, API 34) rồi chạy
`maestro test .maestro/translate-manual.yaml`. **Không** chạy tự động trên push/PR vì flow này
gọi thật tới `groq-relay` → Groq/DeepL/Google Translate free (không mock, tốn quota thật).

### Phạm vi chưa test được

- `AudioRecord`/`MediaProjection` thật, `WakeLock` — cần máy Android thật.
- CodeQL mới quét JavaScript/TypeScript, **chưa** quét Kotlin (cần scaffold cả project Gradle để
  autobuild, để dành làm sau).
- `patch-version.py` tìm `versionName "..."` với dấu ngoặc kép - chưa xác minh 100% khớp với
  template thật do Capacitor 6.x sinh ra - nên build thử 1 lần thật để loại trừ.

## Bảo mật

- **Rate-limit 2 lớp** trong `groq-relay`: theo IP (`request_rate_limit`, đọc từ
  `X-Forwarded-For`) và theo user thật qua JWT ẩn danh (`request_rate_limit_user`) — lớp theo
  user là lớp phòng thủ CHÍNH vì không phụ thuộc header client có thể tự set. `getClientIp()`
  lấy phần tử CUỐI của `X-Forwarded-For` (hop gần Supabase nhất nếu proxy chỉ append), nhưng
  Supabase không cam kết chính thức hành vi này — coi IP lấy được là tín hiệu best-effort (đủ để
  log/điều tra), không phải biên bảo mật cứng. (Fix audit đợt 5: trước đây nếu header chỉ toàn
  khoảng trắng, hàm trả về chuỗi rỗng `""` thay vì `"unknown"` như ý đồ ban đầu — đã sửa: trim
  trước rồi mới quyết định, xem `pure.ts`/`pure.test.ts`.)
- **`sanitizeClientError()` là default-deny** (fix audit đợt 5): chỉ 2 loại message được trả
  nguyên văn ra client — mã đã rút gọn dạng `*_ERROR_<status>:` (vd `GROQ_ERROR_429`), và
  whitelist tường minh 5 message `SERVER_MISSING_*_KEY`/`SERVER_MISSING_AZURE_REGION`. Mọi
  message khác (kể cả message mới phát sinh sau này chưa từng tính tới) trả về `"INTERNAL_ERROR"`
  chung chung — message gốc vẫn được `console.error()` ghi đầy đủ phía server (log Supabase) để
  debug, chỉ không lộ ra client. Trước bản vá này, bất kỳ message không khớp định dạng
  `*_ERROR_<status>:` đều bị trả nguyên văn — có thể lộ thông tin nội bộ ngoài ý muốn.
- **Database**: `request_rate_limit`, `request_rate_limit_user`, `translate_quota_status` đều
  bật Row Level Security và KHÔNG có policy nào — mặc định chặn hết mọi truy cập qua
  `SUPABASE_ANON_KEY` (key này vốn công khai theo thiết kế Supabase, nằm sẵn trong `www/app.js`),
  chỉ Edge Function (qua service role key, tự bypass RLS) mới đọc/ghi được. Cả 3 hàm RPC liên
  quan (`increment_rate_limit` 2 overload, `increment_rate_limit_user`) đều khoá cứng
  `search_path = 'public'` để chặn kiểu tấn công "search path hijacking".
- **Secret**: không có API key nào hardcode trong source — tất cả nạp qua `Deno.env.get()` phía
  backend. `X-App-Secret` ghi trong `app.js`/bundle JS không phải xác thực thật (ai giải nén APK
  cũng đọc được) — chỉ là lớp chặn bot đơn giản, bảo mật thật nằm ở rate-limit + RLS ở trên.
- **WebView**: chỉ tải nội dung local trong `www/`, không có `allowNavigation`/cleartext mở rộng.
  Text từ API ngoài (bản dịch, transcript) luôn qua `escapeHtml()` trước khi chèn `innerHTML`;
  phụ đề video luôn dùng `.textContent`.
- **Xuất file** (nhật ký dịch .txt/.srt, file đã dịch) dùng `@capacitor/filesystem` +
  `@capacitor/share` (`Filesystem.writeFile()` ghi vào `Directory.Cache` rồi `Share.share()` mở
  màn hình chia sẻ/lưu chuẩn của Android) khi chạy trong app native, KHÔNG còn dùng Blob + `<a
  download>` như trước — cách cũ chạy im lặng "thành công" (log vẫn ghi "đã xuất") nhưng thực tế
  không tạo ra file nào, vì Capacitor WebView (`android.webkit.WebView`) không bắt được click vào
  `blob:` URL (không có request mạng nào để `DownloadListener` thấy). Blob + `<a download>` vẫn
  giữ lại làm fallback khi chạy ngoài app (trình duyệt thường lúc dev/test). **Đã xác nhận hoạt
  động đúng trên máy Android thật** (màn hình chia sẻ/lưu mở đúng, file xuất ra được).

## CI/CD (GitHub Actions)

### `android-build.yml` — 3 job

1. **`validation`** — giải nén source + verify checksum, typecheck Deno, chạy Deno test + pytest,
   lint (báo cáo, gồm Detekt cho Kotlin — job này tự setup JDK 17/Temurin riêng cho bước Detekt
   từ fix audit đợt 5 #1, không còn phụ thuộc ngầm vào JDK mặc định của runner). Chạy nhanh,
   không build Android, luôn chạy trên mọi push/PR.
2. **`build-debug`** — phụ thuộc `validation`, chạy unit test Kotlin rồi `assembleDebug`. Chạy
   trên push/PR (hoặc `workflow_dispatch` không tick `build_release`).
3. **`build-release`** — chỉ chạy khi bấm tay `workflow_dispatch` và tick `build_release=true`.
   Chỉ cho phép từ nhánh `main`/`release/*`, yêu cầu đủ 4 secret keystore (thiếu là fail cứng,
   không âm thầm build bằng debug keystore), chạy lại unit test Kotlin trước khi ký, xác thực
   chữ ký APK bằng `apksigner`, tự tạo GitHub Release kèm APK. Tag của Release gồm cả
   `run_number` lẫn `run_attempt` (fix audit đợt 5 #2 — trước đây chỉ có `run_number` nên bấm
   "Re-run failed jobs" sau khi Release đã tạo thành công sẽ luôn fail vì trùng tag).

### `codeql.yml`

Quét bảo mật tĩnh JavaScript/TypeScript (`www/*.js`, `supabase-functions/*.ts`), chạy trên mỗi
PR vào `main` + định kỳ 03:00 UTC thứ Hai hàng tuần.

### `maestro-ui-test.yml`

E2E thủ công trên emulator, xem mục Kiểm thử #5 ở trên. Có khai `permissions: contents: read`
ở cấp job (chỉ cần đọc code, không cần ghi).

### `dependabot.yml`

Chỉ bật 2 ecosystem có manifest thật trong repo: `npm` (gộp PR minor/patch) và `github-actions`,
kiểm tra hàng tuần.

## Secrets cần cấu hình (Settings → Secrets and variables → Actions)

| Secret | Bắt buộc? | Dùng cho |
|---|---|---|
| `ANDROID_KEYSTORE_BASE64` | Bắt buộc (build release) | Keystore ký APK release, base64 |
| `RELEASE_KEYSTORE_PASSWORD` | Bắt buộc (build release) | Mật khẩu keystore |
| `RELEASE_KEY_ALIAS` | Bắt buộc (build release) | Alias key ký |
| `RELEASE_KEY_PASSWORD` | Bắt buộc (build release) | Mật khẩu key |
| `GOOGLE_SERVICES_JSON_BASE64` | Tuỳ chọn | Bật Firebase Crashlytics/Performance/Analytics |
| `RELEASE_CERT_SHA256_FINGERPRINT` | Tuỳ chọn | Tự động đối chiếu fingerprint chứng chỉ ký, chặn build nếu keystore bị đổi nhầm |
| `SLACK_WEBHOOK_URL` | Tuỳ chọn | Thông báo Slack khi build release xong |

Ngoài ra backend `groq-relay` cần các secret riêng đặt ở **Supabase** (không phải GitHub Actions):
`GROQ_API_KEY` (+ `GROQ_API_KEY_2` dự phòng), `GEMINI_API_KEY`, và secret cho DeepL/Azure nếu
dùng các tầng dịch đó.

## Quy trình release

Xem chi tiết từng bước tại [`RELEASE_CHECKLIST.md`](./RELEASE_CHECKLIST.md). Tóm tắt: bump
version trong `package.json` (nếu cần) → kiểm tra đủ 4 secret keystore → đảm bảo đang ở nhánh
`main`/`release/*` và đã có bản debug build xanh → chạy `workflow_dispatch` tick `build_release`
→ (nếu đã bật approval) chờ duyệt → kiểm tra bước xác thực chữ ký + permissions trong log → tải
APK ở tab Actions hoặc tab Releases → cài thử trên máy thật trước khi coi là xong.

### Bật phê duyệt thủ công cho release

Vào repo **Settings → Environments → New environment**, đặt tên đúng `release`, thêm
**Required reviewers**. Sau khi tạo, job `build-release` sẽ tự dừng chờ duyệt trước khi chạy
(GitHub tự ghi log ai đã bấm Approve). Không tạo environment này thì job vẫn chạy bình thường,
chỉ là chưa có bước duyệt thủ công.

### Giám sát (Firebase, tuỳ chọn)

Tạo project Firebase, tải `google-services.json`, encode base64
(`base64 -w0 google-services.json`) rồi lưu vào secret `GOOGLE_SERVICES_JSON_BASE64`. Thiếu
secret này thì bước liên quan trong `build-debug`/`build-release` tự bỏ qua êm, không chặn build.

## Việc còn phải tự làm

- **Test lại khối phát video sau fix lỗi `ytPlayer.loadVideoById is not a function`** — video đã
  xác nhận PHÁT ĐƯỢC thật trên máy, lỗi trên đã sửa (xem mục "Âm lượng video gốc" phía trên). Còn
  cần test: thanh trượt âm lượng chỉnh đúng, bấm "Phát" đổi video MỚI trong lúc player trước chưa
  kịp sẵn sàng (nhánh `pendingVideoIdForPlayer` mới thêm) có load đúng video không, và logic tự
  chuyển hướng fullscreen (nút fullscreen CỦA YouTube, không phải nút "Phóng to" của app) vẫn
  nhận diện đúng sau khi đổi cách kiểm tra.
- **Test `TtsBoostPlugin` trên máy Android thật** (chưa build/chạy thử được, không có Android
  SDK/emulator ở môi trường viết code) — xem đầy đủ ở mục "Tăng âm lượng dịch vượt 100%" phía
  trên: xác nhận `speak()`/`getVoices()` gọi qua Capacitor bridge đúng, đường ≤100% (tts.speak()
  thẳng) VÀ đường >100% (synthesizeToFile → khuếch đại PCM → AudioTrack) đều phát ra tiếng thật,
  không bị dồn ứ hàng đọc khi nghe liên tục nhiều câu, nút 🔇 tắt/bật đúng, danh sách giọng đọc
  tải được (một số máy có thể trả rỗng nếu chưa cài gói giọng TTS nào), và mức vỡ tiếng ở 150%
  có chấp nhận được không.
- Cài thử APK trên máy thật sau mỗi lần release — CI xanh chỉ đảm bảo build/ký thành công, không
  đảm bảo app chạy đúng ý trên máy thật (không có bước UI test tự động nào chạy mặc định).
- Dọn nốt `MagicNumber`/`ReturnCount` trong báo cáo Detekt (artifact `lint-reports-<run>`) — cố
  ý để lại (khác các mục Detekt khác đã dọn, xem mục Kiểm thử #4) vì cần tự quyết ngưỡng/refactor
  hợp lý cho từng chỗ, không nên tự ý đổi hàng loạt.
- Chạy `maestro-ui-test.yml` bằng tay khi cần xác nhận flow dịch qua UI thật (tốn quota API thật,
  không nên tự động hoá định kỳ) — nên theo dõi kỹ lần chạy đầu tiên trên emulator thật.
- Chạy thật `./gradlew testDebugUnitTest` và `pytest -v` (`tests/translate`,
  `tests/cloud_stt_segment`, `tests/auth_token`) trên máy có Android SDK/mạng để xác nhận toàn bộ
  test hiện có build/pass đúng — các test này mới được xác nhận bằng cách gọi tay từng hàm/mô
  phỏng thuật toán, chưa chạy qua `gradlew`/`pytest` thật.
- **Deploy lại `groq-relay`** (`supabase functions deploy groq-relay`) — bắt buộc sau bản vá
  `pure.ts` mới nhất (audit đợt 5: `getClientIp()` trả đúng `"unknown"` cho header toàn khoảng
  trắng, `sanitizeClientError()` đổi sang default-deny). File cũ trên Supabase chưa tự cập nhật
  chỉ vì đã sửa trong repo — phải chạy lệnh deploy tay. Sau khi deploy, theo dõi bảng
  `request_rate_limit` vài ngày đầu để chắc IP ghi nhận hợp lý (không toàn `"unknown"` hay trùng
  lặp bất thường), và thử tạo lỗi thiếu key (`SERVER_MISSING_*_KEY`) 1 lần để xác nhận response
  trả về đúng message whitelist, không còn lộ message lạ nào khác ra client.
- **Test lại tag Release** sau fix audit đợt 5 #2 — chạy `workflow_dispatch` tick `build_release`
  1 lần, bấm "Re-run failed jobs" (giả lập tình huống trước đây bị lỗi `already_exists`) để xác
  nhận tag mới (`run_number-run_attempt`) không còn trùng.
- Repo chưa có `package-lock.json`/`requirements.txt` thật nên cache npm/pip trong CI dùng key
  tạm theo hash file khác — tạo lock file thật (qua Termux hoặc máy khác) sẽ cache chính xác hơn.

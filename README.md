# 🎧 Atun Translator

App Android chạy ngầm: bắt audio đang phát trên máy (YouTube, video, app khác...) hoặc qua
micro, nhận diện giọng nói **offline** bằng [Vosk](https://alphacephei.com/vosk/), dịch sang
tiếng Việt (hoặc ngôn ngữ khác), rồi hiện phụ đề ngay trong app - giống phụ đề trực tiếp cho
video không có sẵn phụ đề Việt.

Đi kèm 1 trang web mini tĩnh (`index.html` / `translate.html`) để tải APK và dịch nhanh văn bản.

## ✨ Tính năng chính

- **Nhận diện giọng nói offline** qua Vosk - không cần mạng, không giới hạn thời lượng nghe
  (cần tải model ngôn ngữ về máy trước, xem mục "Quản lý model ngôn ngữ" bên dưới).
- **Chế độ "Tự động nhận diện"** khi người nói xen kẽ nhiều ngôn ngữ: chạy song song nhiều
  Recognizer Vosk (mỗi ngôn ngữ đã tải 1 bộ), tự chọn kết quả có độ tin cậy cao nhất cho mỗi
  câu (xem `native/MultiLangVoskEngine.kt`). Cần tải trước ít nhất 2 ngôn ngữ. Lúc load nhiều
  model song song, UI hiện progress bar thật ("x/y ngôn ngữ đã sẵn sàng") thay vì text tĩnh.
- **Dịch on-device** qua ML Kit Translate (Google) khi đã tải đủ model 2 chiều nguồn/đích -
  dịch không cần mạng, không giới hạn số lần dịch.
- **Fallback dịch qua mạng** khi thiếu model on-device: ưu tiên Groq (Llama-3.3-70b, chất
  lượng cao, chỉ hỗ trợ Việt/Anh/Trung), lỗi thì tự chuyển sang Google Translate free.
- **Cache bản dịch** theo câu để không gọi mạng lặp lại cho câu đã dịch giống hệt trước đó.
- **Đọc to bản dịch** (Text-to-Speech tiếng Việt có sẵn của Android), tự hạ nhỏ tiếng gốc lúc
  đang đọc (audio ducking) rồi trả lại mức cũ.
- **Web mini** (trình duyệt thu nhỏ ngay trong app, tách biệt hoàn toàn khỏi giao diện chính):
  vào bất kỳ trang web nào để phát audio/video cho tính năng "Bắt âm thanh hệ thống" dịch trực
  tiếp - có nhiều tab, bookmark, lịch sử duyệt web.
- **Phát video (YouTube) ngay trong app kèm phụ đề dịch đè lên video**: kéo-thả đổi vị trí, chỉnh
  cỡ chữ/độ trong suốt nền, và **chỉnh riêng chiều rộng/chiều cao khung phụ đề cho 2 chế độ** -
  video thu nhỏ trong trang lẫn lúc bấm "Phóng to" toàn màn hình (khác tỉ lệ khung hình nên cần 2
  bộ giá trị riêng) - chỉnh bằng 2 cặp thanh trượt Rộng/Cao riêng trong khung cài đặt (thường /
  toàn màn hình), xem trước ngay trên khung video hiện tại (chưa mở fullscreen thật được từ
  trang cài đặt nên tỉ lệ preview có thể lệch chút so với lúc bấm "Phóng to" thật). Mọi cài đặt
  lưu vào máy, giữ nguyên cho lần mở app sau.
- **Nhật ký dịch** trong app: xem lại, tìm kiếm, xuất file `.srt`, `.vtt`, hoặc `.txt` song ngữ.
- **Ghi log lỗi/crash cục bộ** + nút chia sẻ log, và ghi nhận % pin hao mỗi phiên nghe - số
  liệu này giờ cũng hiện trực tiếp ngay trên UI chính (không cần bấm "Chia sẻ log" mới thấy),
  tự cập nhật mỗi 30s trong lúc đang nghe.
- **Quản lý model dễ hơn**: nút "Huỷ tải" cho model đang tải dở (không cần đợi hết hoặc tắt
  app), cảnh báo dung lượng trống trước khi tải cả model Vosk lẫn model dịch ML Kit.
- **Onboarding tối giản**: chỉ hiện đúng bước đang cần (quyền thông báo → miễn trừ tối ưu pin),
  ẩn hết khi đã xong.

## 🌐 Quản lý model ngôn ngữ

| Model | Cách có được | Ghi chú |
|---|---|---|
| Vosk **English**, **English (chính xác cao)**, **Tiếng Việt**, **日本語**, **한국어**... (nhận diện giọng nói) | **Tải on-demand** | Bấm tải trong app, lấy trực tiếp từ [alphacephei.com/vosk/models](https://alphacephei.com/vosk/models) (có mirror dự phòng trên Hugging Face nếu nguồn gốc chập chờn), giải nén vào bộ nhớ riêng của app. Có thể huỷ giữa chừng bằng nút "Huỷ". |
| ML Kit **Translate** (mọi ngôn ngữ: vi/en/ja/ko/zh...) | **Tải on-demand** | Bấm tải trong mục "Quản lý mô hình dịch" - mỗi ngôn ngữ ~30MB, ML Kit tự ghép cặp qua tiếng Anh khi cần, có cảnh báo dung lượng trống trước khi tải |

Nói cách khác: **không còn model nào đóng gói sẵn trong APK nữa** - kể cả tiếng Anh (trước đây
bundle sẵn) giờ cũng ở dạng **tải về theo yêu cầu (on-demand)** như mọi ngôn ngữ khác. Lý do
đổi: các file model Vosk (kể cả bản "small") có file nhị phân rất lớn
(`Gr.fst`/`HCLr.fst`/`final.mdl`/`final.ie`), trước đây phải cắt mảnh `.partNNN` mới commit/tải
lên được qua giao diện GitHub trên di động, rất bất tiện mỗi lần cập nhật model. Đổi lại: APK
nhẹ hơn nhiều và **repo không còn file lớn nào phải cắt mảnh nữa** - nhưng người dùng phải bấm
tải ít nhất 1 model nhận diện giọng nói (khuyên dùng English, ~40MB) trước khi bắt đầu nghe lần
đầu, không có sẵn ngay sau khi cài như trước.

File model được tải thẳng từ nguồn chính chủ (alphacephei.com / Google ML Kit) về bộ nhớ máy,
không cần cài lại APK khi muốn dùng thêm ngôn ngữ mới.

## 🏗️ Kiến trúc / cấu trúc thư mục

```
native/               Plugin Capacitor viết bằng Kotlin
  MainActivity.kt          Đăng ký toàn bộ plugin
  AudioCaptureService.kt   Bắt audio hệ thống/micro chạy nền (Foreground Service)
  AudioCapturePlugin.kt    Cầu nối JS <-> AudioCaptureService
  SttEngine.kt              Interface chung cho VoskEngine/MultiLangVoskEngine
  VoskEngine.kt             Nhận diện giọng nói qua Vosk (1 ngôn ngữ)
  MultiLangVoskEngine.kt    Chế độ "Tự động nhận diện" - chạy song song nhiều VoskEngine
  VadDetector.kt            Voice Activity Detection (phát hiện có giọng nói hay im lặng)
  ModelManagerPlugin.kt    Tải/xoá/huỷ tải model Vosk on-demand (kể cả tiếng Anh)
  MlkitTranslatePlugin.kt  Dịch on-device qua ML Kit Translate
  OverlayWindowPlugin.kt / OverlayView.kt   (đã ngừng dùng chữ nổi đè app khác, còn giữ code)
  ScreenOrientationPlugin.kt  Khoá/mở khoá xoay ngang màn hình lúc fullscreen video trong app
  TtsPlugin.kt             Đọc to bản dịch (Text-to-Speech)
  ExportPlugin.kt          Xuất/chia sẻ file (log lỗi, .srt/.vtt/.txt) qua Share Sheet của Android
  AppLog.kt                Log lỗi/crash cục bộ, dùng chung cho crash handler + ExportPlugin
  BrowserActivity.kt / BrowserStorage.kt / BrowserLauncherPlugin.kt
                            "Web mini" - trình duyệt thu nhỏ, Activity RIÊNG tách biệt hoàn
                            toàn khỏi www/index.html, để vào web bất kỳ phát audio/video

www/                  Giao diện app (chạy trong Capacitor WebView)
  index.html               Toàn bộ UI + logic chính
  src/services/             Bridge JS <-> plugin native (audio, translation, export, tts)

scripts/              Script Python patch Gradle/Manifest lúc build CI (chạy trong workflow)
supabase-functions/   Edge Function: groq-relay (dịch), youtube-extract
tests/                Test harness Python (pytest) - xem mục Test bên dưới

index.html / translate.html / style.css   Trang web mini tĩnh đi kèm (GitHub Pages) - KHÁC
                      với "Web mini" trong app (BrowserActivity) ở trên: trang này chạy trên
                      máy tính/GitHub Pages, chỉ để tải APK + dịch nhanh văn bản.
                      (Không nằm trong các part zip source app - deploy riêng qua Pages.)
```

## 📥 Build & cài đặt

APK được build tự động bằng GitHub Actions (`.github/workflows/build-android-apk.yml`) mỗi khi
push lên nhánh `main`:

1. Unzip source vào repo root - workflow tự nhận cả 2 kiểu file commit ở repo root:
   `atun-translator-source.zip` (kiểu hiện tại - toàn bộ source nén chung 1 file duy nhất, vì
   GitHub di động không hỗ trợ tải cả thư mục qua giao diện web) hoặc `atun-translator-part*.zip`
   (kiểu cũ - chia nhiều phần, vẫn được hỗ trợ song song nếu cần dùng lại). Từ khi bỏ model Vosk
   bundle sẵn, toàn bộ source đã rất nhẹ (dưới 200KB), nên giờ thường chỉ cần đúng 1 file
   `atun-translator-source.zip` là đủ, không cần cắt mảnh `.partNNN` như trước nữa.
2. Chạy test harness (`tests/vad`, `tests/translate`, `tests/model_catalog`) - fail sớm nếu
   code/catalog có lỗi rõ ràng, trước khi tốn thời gian build Android thật.
3. Patch Gradle/Manifest, copy native Kotlin sources vào project Android tạo bởi Capacitor.
4. Build cả bản **debug** và **release** (minify + ký bằng keystore riêng nếu đã cấu hình
   secret `ANDROID_KEYSTORE_BASE64`, hoặc tự fallback debug keystore).
5. Tạo/cập nhật 1 **GitHub Release** tên `latest` kèm sẵn `atun-translator.apk` - cho link tải
   công khai cố định, không cần đăng nhập GitHub:
   ```
   https://github.com/<user>/<repo>/releases/latest/download/atun-translator.apk
   ```

Cài lên máy: tải file `.apk` từ link trên → mở file → bật "Cho phép cài từ nguồn này" nếu máy
chặn → cài xong mở app, cấp đủ quyền theo hướng dẫn onboarding trong app → vào mục **"Quản lý
mô hình ngôn ngữ (STT)"** bấm tải ít nhất 1 model (khuyên dùng English) trước khi bắt đầu nghe -
app không còn model nào có sẵn ngay sau khi cài.

> ⚠️ Trang `index.html` (web mini tĩnh, GitHub Pages) đang trỏ link tải tới
> `REPLACE_ME_user/REPLACE_ME_repo` - cần sửa lại đúng username/repo GitHub của bạn 1 lần
> trước khi dùng trang này.

## 🧪 Test harness (chạy trên máy tính, không cần Android)

```bash
cd tests/vad && pip install pytest && pytest -v            # VadDetector
cd tests/translate && pip install pytest && pytest -v      # Retry/backoff dịch Google Translate free
cd tests/model_catalog && pip install pytest && pytest -v  # Validate CATALOG model không có URL placeholder
```

Các file `.py` là bản dịch tay từ logic Kotlin/JS gốc - xem `README.md` trong từng thư mục con
của `tests/` để biết chi tiết và lưu ý giữ đồng bộ khi sửa code gốc.

## 📝 Ghi chú khác

- Log lỗi/crash + số liệu hao pin theo phiên được lưu cục bộ trên máy, và giờ hiện trực tiếp
  ngay trên UI chính (không cần mở khung debug) - bấm "📤 Chia sẻ log lỗi" trong khung debug
  của app để xuất ra khi cần báo lỗi.
- Dịch qua mạng (Groq/Google Translate free) chỉ dùng khi thiếu model on-device tương ứng -
  không cần API key phía người dùng.
- `native/OverlayView.kt` / `OverlayWindowPlugin.kt` vẫn còn trong source (đã ngừng dùng tính
  năng chữ nổi đè app khác nhưng chưa xoá code) - nếu muốn dọn hẳn, cần gỡ đăng ký plugin ở
  `MainActivity.kt`, gỡ dòng copy tương ứng trong workflow, và kiểm tra `scripts/patch-manifest.py`
  có patch gì riêng cho quyền `SYSTEM_ALERT_WINDOW` không.

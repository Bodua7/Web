# 🎧 Atun Translator

App Android chạy ngầm: bắt audio đang phát trên máy (YouTube, video, app khác...) hoặc qua
micro, nhận diện giọng nói **offline** bằng [Vosk](https://alphacephei.com/vosk/), dịch sang
tiếng Việt (hoặc ngôn ngữ khác), rồi hiện phụ đề ngay trong app - giống phụ đề trực tiếp cho
video không có sẵn phụ đề Việt.

Đi kèm 1 trang web mini tĩnh (`index.html` / `translate.html` / `log.html`) để tải APK, dịch
nhanh văn bản, và xem lại nhật ký dịch đã đồng bộ từ điện thoại trên màn hình lớn.

## ✨ Tính năng chính

- **Nhận diện giọng nói offline** qua Vosk - không cần mạng, không giới hạn thời lượng nghe.
- **Dịch on-device** qua ML Kit Translate (Google) khi đã tải đủ model 2 chiều nguồn/đích -
  dịch không cần mạng, không giới hạn số lần dịch.
- **Fallback dịch qua mạng** khi thiếu model on-device: ưu tiên Groq (Llama-3.3-70b, chất
  lượng cao, chỉ hỗ trợ Việt/Anh/Trung), lỗi thì tự chuyển sang Google Translate free.
- **Cache bản dịch** theo câu để không gọi mạng lặp lại cho câu đã dịch giống hệt trước đó.
- **Đọc to bản dịch** (Text-to-Speech tiếng Việt có sẵn của Android), tự hạ nhỏ tiếng gốc lúc
  đang đọc (audio ducking) rồi trả lại mức cũ.
- **Nhật ký dịch** trong app: xem lại, tìm kiếm, xuất file `.srt` hoặc `.txt` song ngữ.
- **Đồng bộ nhiều máy** qua "Mã đồng bộ" riêng (không cần đăng nhập) - đồng bộ settings +
  nhật ký dịch giữa các thiết bị, xem lại trên `log.html` ở máy tính.
- **Ghi log lỗi/crash cục bộ** + nút chia sẻ log, và ghi nhận % pin hao mỗi phiên nghe để theo
  dõi độ ổn định/hao pin thực tế qua thời gian.
- **Onboarding tối giản**: chỉ hiện đúng bước đang cần (quyền thông báo → miễn trừ tối ưu pin),
  ẩn hết khi đã xong.

## 🌐 Quản lý model ngôn ngữ

| Model | Cách có được | Ghi chú |
|---|---|---|
| Vosk **English** (nhận diện giọng nói) | **Bundle sẵn trong APK** | Dùng ngay, không cần tải |
| Vosk **English (chính xác cao)**, **Tiếng Việt**, **日本語**, **한국어** | **Tải on-demand** | Bấm tải trong app, lấy trực tiếp từ [alphacephei.com/vosk/models](https://alphacephei.com/vosk/models), giải nén vào bộ nhớ riêng của app |
| ML Kit **Translate** (mọi ngôn ngữ: vi/en/ja/ko/zh...) | **Tải on-demand** | Bấm tải trong mục "Quản lý mô hình dịch" - mỗi ngôn ngữ ~30MB, ML Kit tự ghép cặp qua tiếng Anh khi cần |

Nói cách khác: **chỉ tiếng Anh (Vosk) được đóng gói sẵn trong APK** để giữ dung lượng cài đặt
nhỏ; **tất cả ngôn ngữ còn lại** (cả nhận diện giọng nói lẫn dịch) đều ở dạng **tải về theo
yêu cầu (on-demand)** - người dùng tự bấm tải khi cần, file model được tải thẳng từ nguồn
chính chủ (alphacephei.com / Google ML Kit) về bộ nhớ máy, không cần cài lại APK khi muốn dùng
thêm ngôn ngữ mới.

## 🏗️ Kiến trúc / cấu trúc thư mục

```
native/               Plugin Capacitor viết bằng Kotlin
  MainActivity.kt          Đăng ký toàn bộ plugin
  AudioCaptureService.kt   Bắt audio hệ thống/micro chạy nền (Foreground Service)
  AudioCapturePlugin.kt    Cầu nối JS <-> AudioCaptureService
  VoskEngine.kt            Nhận diện giọng nói qua Vosk
  VadDetector.kt           Voice Activity Detection (phát hiện có giọng nói hay im lặng)
  ModelManagerPlugin.kt    Tải/xoá model Vosk on-demand
  MlkitTranslatePlugin.kt  Dịch on-device qua ML Kit Translate
  OverlayWindowPlugin.kt / OverlayView.kt   (đã ngừng dùng chữ nổi đè app khác)
  TtsPlugin.kt             Đọc to bản dịch (Text-to-Speech)
  ExportPlugin.kt          Xuất/chia sẻ file (log lỗi, .srt, .txt) qua Share Sheet của Android
  AppLog.kt                Log lỗi/crash cục bộ, dùng chung cho crash handler + ExportPlugin

www/                  Giao diện app (chạy trong Capacitor WebView)
  index.html               Toàn bộ UI + logic chính
  src/services/             Bridge JS <-> plugin native (audio, translation, export, tts)

scripts/              Script Python patch Gradle/Manifest lúc build CI (chạy trong workflow)
supabase-functions/   Edge Function: groq-relay (dịch), translator-sync (đồng bộ đa máy)
tests/                Test harness Python (pytest) - xem mục Test bên dưới
assets-for-android/   Model Vosk tiếng Anh bundle sẵn trong APK

index.html / translate.html / log.html / style.css   Trang web mini đi kèm (GitHub Pages)
```

## 📥 Build & cài đặt

APK được build tự động bằng GitHub Actions (`.github/workflows/build-apk.yml`) mỗi khi push
lên nhánh `main`:

1. Unzip các file `atun-translator-part*.zip` vào repo root (source được chia nhỏ để tải lên
   dễ hơn).
2. Patch Gradle/Manifest, copy native Kotlin sources vào project Android tạo bởi Capacitor.
3. Build cả bản **debug** và **release** (minify + ký bằng keystore riêng nếu đã cấu hình
   secret `ANDROID_KEYSTORE_BASE64`, hoặc tự fallback debug keystore).
4. Tạo/cập nhật 1 **GitHub Release** tên `latest` kèm sẵn `atun-translator.apk` - cho link tải
   công khai cố định, không cần đăng nhập GitHub:
   ```
   https://github.com/<user>/<repo>/releases/latest/download/atun-translator.apk
   ```

Cài lên máy: tải file `.apk` từ link trên → mở file → bật "Cho phép cài từ nguồn này" nếu máy
chặn → cài xong mở app, cấp đủ quyền theo hướng dẫn onboarding trong app.

> ⚠️ Trang `index.html` (web mini) đang trỏ link tải tới `REPLACE_ME_user/REPLACE_ME_repo` -
> cần sửa lại đúng username/repo GitHub của bạn 1 lần trước khi dùng trang này.

## 🔄 Đồng bộ nhiều máy

Vào mục "Đồng bộ nhiều máy" trong app, đặt 1 "Mã đồng bộ" riêng (8-64 ký tự chữ/số/-/_, giống
mật khẩu), gõ **giống nhau** trên các máy muốn đồng bộ. Bấm "Đẩy lên cloud" / "Tải về từ cloud"
để đồng bộ settings + nhật ký dịch. Mã đồng bộ được hash trước khi lưu (không lưu plaintext) và
có rate-limit chống đoán mã (xem `supabase-functions/translator-sync/index.ts`).

Xem lại nhật ký đã đồng bộ trên máy tính tại `log.html` - nhập đúng mã đồng bộ để tải và tìm
kiếm/xuất file dễ hơn so với cuộn trên điện thoại.

## 🧪 Test harness (chạy trên máy tính, không cần Android)

```bash
cd tests/vad && pip install pytest && pytest -v            # VadDetector
cd tests/translate && pip install pytest && pytest -v      # Retry/backoff dịch Google Translate free
cd tests/model_catalog && pip install pytest && pytest -v  # Validate CATALOG model không có URL placeholder
```

Các file `.py` là bản dịch tay từ logic Kotlin/JS gốc - xem `README.md` trong từng thư mục con
của `tests/` để biết chi tiết và lưu ý giữ đồng bộ khi sửa code gốc.

## 📝 Ghi chú khác

- Log lỗi/crash + số liệu hao pin theo phiên được lưu cục bộ trên máy - bấm "📤 Chia sẻ log
  lỗi" trong khung debug của app để xuất ra khi cần báo lỗi.
- Dịch qua mạng (Groq/Google Translate free) chỉ dùng khi thiếu model on-device tương ứng -
  không cần API key phía người dùng.

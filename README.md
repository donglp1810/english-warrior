![Spec](https://github.com/donglp1810/english-warrior/blob/main/spec_1_img.PNG)
# Design Spec: Core Học Nghe-Nói (Slice 1) — English Warrior

> Ngày: 2026-06-22 · Trạng thái: Đã duyệt (qua brainstorming) · Sub-project: #1

## Context

`English Warrior` là ý tưởng app học tiếng Anh kết hợp game (vòng lặp Học ↔ Game thủ thành).
Toàn bộ ý tưởng là một product lớn gồm ~6 hệ thống độc lập (core học, gamification/pet, game
Unity, cầu nối kinh tế, social/team, phụ huynh, backend). Cố viết 1 spec cho tất cả sẽ không
khả thi, nên ta **chia nhỏ** và làm từng sub-project. Tài liệu này chỉ đặc tả **sub-project #1:
Core Học Nghe-Nói**, nền móng cho mọi thứ sau này.

**Mục tiêu slice này:** một vòng lặp học Nghe–Nói hoàn chỉnh, chạy được, local-only:
nghe câu mẫu (TTS, chỉnh tốc độ) → đọc lại → ghi âm + STT on-device chấm khớp → hiện kết quả →
lên lịch ôn theo spaced repetition → nhắc ôn bằng local notification. Cấp độ nội dung tăng dần
từ **từ → cụm → câu → đoạn**.

### Quyết định đã chốt với người dùng
- **Nền tảng:** cả Android (Kotlin/Jetpack Compose) **và** iOS (Swift/SwiftUI), làm song song.
- **Kiến trúc (Phương án A):** hai app native **độc lập**, dùng chung **file nội dung JSON** +
  **bản đặc tả thuật toán** (không dùng KMP). Logic viết 2 lần nhưng spec hoá chính xác + có
  bộ test vector chung để 2 bên cho kết quả giống nhau.
- **STT:** on-device, miễn phí — Android `SpeechRecognizer` (offline), iOS `SFSpeechRecognizer`
  (`requiresOnDeviceRecognition = true`).
- **Nội dung:** bài học soạn sẵn dạng dữ liệu (JSON) + audio mẫu sinh bằng **TTS runtime
  on-device** (Android `TextToSpeech`, iOS `AVSpeechSynthesizer`) — hỗ trợ chỉnh tốc độ, miễn phí.
- **Lưu trữ:** local-only (Android Room, iOS SwiftData). Không backend, không tài khoản.
- **Spaced repetition reminder:** CÓ, qua local notification.
- **Ngoài scope slice này:** pet/Teddy, animation/gamification, hero, game Unity, social/team,
  phụ huynh, backend/đồng bộ, AI pronunciation feedback.

### Giới hạn quan trọng cần ghi rõ (đã được người dùng chấp nhận)
STT on-device chỉ trả về **văn bản nhận dạng** (transcript), không chấm phoneme. Vì vậy việc
"chấm phát âm" ở slice này thực chất là **so khớp xem máy có *nghe ra* đúng các từ mục tiêu
không** — gần đúng với độ rõ ràng/phát âm, KHÔNG phải đánh giá chất lượng phát âm thật sự.
Đây là đánh đổi có chủ đích khi bỏ AI. Tài liệu/UX cần phản ánh đúng (tránh hứa "chấm phát âm chuẩn").

---

## Kiến trúc tổng thể

```
english_warrior/
├─ content/                 # Nguồn chân lý NỘI DUNG (dùng chung 2 app)
│  ├─ schema/lesson.schema.json
│  └─ lessons/*.json        # bài học soạn sẵn
├─ docs/
│  └─ superpowers/specs/    # design doc + ĐẶC TẢ THUẬT TOÁN + test vectors
│     ├─ 2026-06-22-core-listen-speak-design.md   (file này)
│     └─ algorithm-spec.md  # SRS + scoring: pseudocode + golden test vectors
├─ android/                 # app Android độc lập (Kotlin/Compose/Room)
└─ ios/                     # app iOS độc lập (Swift/SwiftUI/SwiftData)
```

Hai app **không** chia sẻ code. Chúng chia sẻ: (1) định dạng JSON nội dung, (2) bản đặc tả thuật
toán + bộ **golden test vectors** đặt trong `docs/.../algorithm-spec.md`, để unit test 2 bên xác
nhận cho cùng output.

---

## Đặc tả thành phần

### 1. Định dạng nội dung bài học (JSON, dùng chung)
Một `Lesson` chứa metadata + danh sách `items`. Mỗi item là một đơn vị luyện (từ/cụm/câu/đoạn).

```jsonc
// content/lessons/lesson_l3_greetings.json (ví dụ)
{
  "id": "l3_greetings",
  "title": "Daily Greetings",
  "level": "SENTENCE",            // WORD | PHRASE | SENTENCE | PARAGRAPH
  "order": 12,                     // thứ tự mở khoá
  "items": [
    {
      "id": "l3_greetings_001",
      "text": "How are you doing today?",
      "ipa": "haʊ ɑːr juː ˈduːɪŋ təˈdeɪ",   // optional, để hiển thị
      "ttsLocale": "en-US"
    }
  ]
}
```
- `level` quyết định ngưỡng chấm điểm và tốc độ TTS gợi ý.
- Lộ trình tăng dần: lesson sắp theo `level` + `order`; mở khoá tuần tự (xem §5).
- Audio KHÔNG nhúng file — sinh runtime bằng TTS từ `text` (chỉnh được tốc độ).

### 2. Phát mẫu bằng TTS (chỉnh tốc độ — "tốc độ nghe")
- 3 mức tốc độ: **0.5x / 0.75x / 1.0x** (map sang `setSpeechRate` Android, `AVSpeechUtterance.rate` iOS).
- Mức mặc định theo level (đoạn dài → mặc định chậm hơn). User đổi được trong màn học + Settings.
- Android: `TextToSpeech`, locale `en-US`, kiểm tra/khởi tạo engine, fallback nếu thiếu voice.
- iOS: `AVSpeechSynthesizer` + `AVSpeechUtterance` (rate, voice `en-US`).

### 3. Ghi âm + STT on-device + chấm khớp
Luồng 1 item: hiện text → (tuỳ chọn) nghe mẫu → nhấn ghi âm → STT → chấm → kết quả → retry/next.

**STT:**
- Android: `SpeechRecognizer` với `EXTRA_PREFER_OFFLINE = true`, `EXTRA_LANGUAGE = en-US`;
  lấy `results` + `confidenceScores`. Xử lý case engine offline chưa cài → hướng dẫn user.
- iOS: `SFSpeechRecognizer(locale: en-US)`, `requiresOnDeviceRecognition = true`,
  `SFSpeechAudioBufferRecognitionRequest` + `AVAudioEngine` tap. Xin quyền Speech + Microphone.
- (Tuỳ chọn) lưu file ghi âm để user nghe lại — không bắt buộc slice này.

**Thuật toán chấm điểm (đặc tả chính xác, viết trong `algorithm-spec.md`):**
1. Chuẩn hoá cả `target` và `transcript`: lowercase, bỏ dấu câu, gộp khoảng trắng, trim.
2. Tách token theo từ.
3. So khớp bằng **word-level Levenshtein** trên chuỗi token → đếm số token mục tiêu khớp đúng
   (matched = targetLen − (substitutions + deletions trên target)).
4. `score = matched / targetTokenCount` (0.0–1.0).
5. **Ngưỡng đậu theo level** (cấu hình được): WORD = 1.0, PHRASE = 0.8, SENTENCE = 0.75,
   PARAGRAPH = 0.7.
6. Kết quả trả về: `score`, `passed`, danh sách từ **khớp / sai / thiếu** để highlight UI.
- Phải kèm **golden test vectors** (cặp target/transcript → score/passed mong đợi) để Android &
  iOS test ra cùng kết quả → chống lệch logic giữa 2 app.

### 4. Mô hình dữ liệu local (mỗi app tự định nghĩa, schema giống nhau)
- `Lesson`, `LessonItem` — nạp từ JSON khi cài/cập nhật (seed vào DB hoặc đọc trực tiếp).
- `ReviewState` (1–1 với item): `box` (Leitner), `dueDate`, `lastScore`, `streak`, `totalAttempts`.
- `Attempt`: `itemId`, `timestamp`, `transcript`, `score`, `passed`.
- `Settings`: tốc độ TTS mặc định, giờ nhắc ôn, ngưỡng (nếu cho chỉnh).
- Android: Room entities + DAO. iOS: SwiftData models (`@Model`).

### 5. Spaced Repetition (Leitner — deterministic, dễ test cùng kết quả)
Chọn **Leitner boxes** thay vì SM-2: đơn giản, xác định, dễ đảm bảo 2 app giống nhau, hợp HS-SV.
- 5 hộp với khoảng cách: **Box1=0 ngày (làm ngay), Box2=1, Box3=3, Box4=7, Box5=14**.
- Item **đậu** → lên 1 hộp (tối đa Box5); **trượt** → về Box1.
- `dueDate = ngày hôm nay + interval(box)`. Item mới bắt đầu ở Box1.
- **Hàng đợi hôm nay** = các item có `dueDate <= hôm nay`, ưu tiên hộp thấp + quá hạn lâu.
- Mở khoá lesson: hoàn thành (đậu lần đầu) đủ tỉ lệ item của lesson trước → mở lesson `order` kế.
- Pseudocode + test vectors viết trong `algorithm-spec.md`.

### 6. Nhắc ôn — Local Notification
- Tính số item `due` cho hôm nay; lên lịch thông báo tại **giờ user chọn** (Settings, mặc định 19:00).
- Nội dung: "Bạn có N mục cần ôn hôm nay 📚".
- Android: `WorkManager`/`AlarmManager` + `NotificationCompat`; xin quyền `POST_NOTIFICATIONS` (API 33+).
- iOS: `UNUserNotificationCenter` local notification; xin quyền notification.
- Lên lại lịch mỗi khi học xong / mở app (vì local-only, không có push server).

### 7. Màn hình (tối thiểu, mỗi app native)
1. **Home / Hôm nay:** số item due + nút "Ôn ngay"; danh sách lesson theo level (khoá/mở).
2. **Lesson Player:** hiển thị item (text + IPA) → nghe mẫu (nút tốc độ) → ghi âm → màn kết quả
   (score, từ đúng/sai highlight, nút Retry / Next).
3. **Settings:** tốc độ TTS mặc định, giờ nhắc ôn, (tuỳ chọn) reset tiến trình.
4. (Tuỳ chọn) **Progress:** streak, số item theo từng hộp.

### 8. Quyền & cấu hình
- Microphone (cả 2), Speech recognition (iOS `NSSpeechRecognitionUsageDescription`),
  Notifications (cả 2), `NSMicrophoneUsageDescription` (iOS).
- Android `minSdk` đề xuất 26+ (TTS/STT/Notification ổn định).

---

## Phương án thay thế đã cân nhắc (không chọn)
- **KMP (Phương án B):** chia sẻ logic SRS/scoring 1 lần → không lệch. Đã loại theo lựa chọn của
  người dùng (muốn 2 app native thuần). Đổi lại, ta dùng **golden test vectors chung** để bù rủi ro lệch.
- **Cross-platform (Flutter/RN):** trái yêu cầu native — loại.
- **SM-2 cho SRS:** thích nghi tốt hơn nhưng khó đảm bảo 2 app giống hệt + phức tạp hơn cần thiết
  cho slice này — để dành nâng cấp sau.

---

## Verification
- **Unit test (bắt buộc):** scoring + Leitner trên Android (JUnit) và iOS (XCTest) chạy **cùng
  bộ golden test vectors** → cả hai phải khớp giá trị mong đợi. Đây là chốt chống lệch logic.
- **Thủ công E2E (thiết bị thật, vì mic/STT):**
  1. Mở 1 lesson → nghe mẫu ở 0.5x/0.75x/1.0x (kiểm tra tốc độ đổi đúng).
  2. Đọc đúng → `passed`, item lên hộp, `dueDate` đẩy theo interval.
  3. Đọc sai/thiếu từ → `failed`, từ sai/thiếu được highlight, item về Box1.
  4. Hoàn thành đủ item lesson → lesson kế mở khoá.
  5. Đặt giờ nhắc gần hiện tại → nhận local notification đúng giờ với số item due đúng.
  6. Tắt mạng → app vẫn học được (xác nhận on-device thật sự offline).
- **Đa thiết bị:** thử ít nhất 1 Android + 1 iOS để bắt khác biệt voice/engine STT.

---

## Ghi chú quy trình
- Đây là sub-project ĐẦU TIÊN. Sau khi hoàn tất, các sub-project kế (pet/gamification → game Unity
  → cầu nối kinh tế → social → phụ huynh → backend) sẽ có spec/plan riêng.
- Bước kế tiếp: dùng skill `writing-plans` để bóc tách kế hoạch triển khai chi tiết theo các mốc.

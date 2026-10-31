# Rà soát mức độ hoàn thiện và rủi ro — 31/10/2026

Phạm vi: commit `62b96bd`, toàn bộ server, Python TTS worker, giao diện live,
admin, particle engine, dependency lock và spec. Đây là review code và kiểm thử
cục bộ; chưa xác nhận kết nối phòng TikTok thật hay phát qua OBS.

## Kết luận

Ứng dụng là **overlay tương tác 9:16 dùng với OBS/TikTok Live Studio**, không phải
phần mềm tự phát video lên TikTok. Có đủ lõi để demo: JSON-RPC HTTP/WebSocket,
nhận sự kiện TikTok, bảy nghi thức, bảng điểm lưu JSON, hiệu ứng canvas/sprite và
giọng đọc ZeroTTS. Chưa đủ tin cậy để mở public hoặc chạy livestream dài có quà
thật mà không giám sát. Không có automated test suite, CI, supervisor hay cấu
hình triển khai trong repo.

Luồng xử lý: TikTok hoặc admin → `server.js` → cộng điểm/lưu trạng thái → broadcast
WebSocket → `public/js/live.js` vẽ và gọi TTS → Python tổng hợp WAV → trình duyệt
phát âm thanh. OBS chỉ cần capture trang `/?obs=1` ở 1080 × 1920.

## Phát hiện cần ưu tiên

### R1 — Cao: toàn bộ quyền quản trị không có xác thực

- Vị trí: `server.js:1042–1087,1096–1100,1107–1140,1148–1207`.
- Bằng chứng: `app.use(cors())`; HTTP RPC và WebSocket đều gọi thẳng dispatcher;
  `server.listen(PORT, callback)` không chỉ định localhost. Không kiểm tra token,
  Origin hay vai trò trước reset bảng điểm, sửa settings, đọc TTS, đổi phòng live.
- Tác động: bên truy cập được cổng có thể điều khiển sân khấu, reset điểm, gây
  tải TTS hoặc thay kênh. WebSocket cũng không kiểm tra Origin. Khả năng khai thác
  từ mạng phụ thuộc firewall; khả năng từ website khác còn phụ thuộc chính sách
  bảo vệ mạng local của trình duyệt, không nên dựa vào đó.
- Đề xuất: mặc định bind `127.0.0.1`, tách quyền đọc overlay và điều khiển, token
  cho mutation, allowlist Origin và giới hạn tốc độ/body/WS payload.
- Giảm thiểu khi thử: launcher chỉ bind localhost; không public tunnel/port.

### R2 — Cao: SSRF qua đầu vào kết nối TikTok

- Vị trí: `server.js:794–816,1079–1080`.
- Bằng chứng: URL bất kỳ bắt đầu `http://` hoặc `https://` được `fetch` với
  `redirect: 'follow'`, không allowlist host, giới hạn địa chỉ mạng hay timeout.
- Tái hiện cô lập: `resolveTikTokUsername('http://127.0.0.1:4567/internal')`
  gọi đúng URL đó trong fetch stub. Không gửi request tấn công tới dịch vụ thật.
- Tác động: người gọi RPC có thể khiến máy chạy server truy cập URL nội bộ;
  không khẳng định đọc được nội dung phản hồi vì resolver không trả body.
- Đề xuất: ưu tiên nhận username; chỉ nhận link HTTPS TikTok hợp lệ, kiểm tra
  từng redirect, chặn IP nội bộ và thêm timeout.

### R3 — Cao: đầu vào RPC có thể làm hỏng bảng điểm và prototype

- Vị trí: `server.js:392–423,576–600,666–670`.
- Bằng chứng: dictionary `state.devotees` là object thường, tra bằng `uniqueId`
  mà không kiểm tra own-property; `coins` chỉ qua `Number()` và `Math.max`.
- Tái hiện trong VM độc lập: `uniqueId: '__proto__'` sửa `Object.prototype.name`
  và khiến `Object.prototype.merit` thành NaN; `coins: 'Infinity'` làm tổng điểm
  thành Infinity, serialize JSON thành `null`. `uniqueId` dạng object gây lỗi
  `.slice is not a function`. Không gửi payload này vào dữ liệu live.
- Tác động: sai điểm, trạng thái không hữu hạn và tác động tới object dùng chung.
  Chưa xác nhận thực thi mã hay nâng quyền từ lỗi này.
- Đề xuất: schema từng method, số nguyên hữu hạn có trần, giới hạn độ dài và
  kiểu; dùng Map/object không prototype và kiểm tra own-property.

### R4 — Trung bình: quà combo bị cộng lại và luật xu lệch README

- Vị trí: `server.js:580–593,932–965`.
- Tái hiện: hai GIFT giống nhau cùng group, repeatCount=1, repeatEnd=false cộng
  tổng 40 điểm thay vì 20. `Math.max(1, repeatCount-prevCount)` biến delta=0
  thành 1; xử lý chống trùng hiện chỉ áp dụng một nhánh repeatEnd.
- Quà 5 xu được 100 điểm; README mô tả 5–99 xu ×30, tức 150 điểm. Code dùng
  ngưỡng ×30 từ 10 xu. LIKE cũng bị cap 15 mỗi event (`server.js:907`), có thể
  thấp hơn số tim thật khi TikTok gửi một đợt lớn.
- Đề xuất: delta phải dương, xét đúng loại gift/group/msg ID, xóa tracker khi
  kết thúc combo, chốt lại bảng luật và thêm fixture sự kiện thực tế.
- Không báo lỗi đếm đôi FOLLOW/SHARE: connector **2.5.0 trong lock hiện tại**
  route FOLLOW/SHARE riêng, không đồng thời emit SOCIAL cho cùng nhánh.

### R5 — Trung bình: lưu dữ liệu và phục hồi chưa phù hợp phiên dài

- Vị trí: `server.js:298–325,332–335,892–895`.
- `saveState()` debounce 1,2 giây; dòng tương tác liên tục dưới khoảng đó có
  thể dời lần ghi vô hạn. Ghi trực tiếp vào file chính, không atomic rename,
  không flush khi shutdown, không backup. MEMBER cộng điểm nhưng không gọi save.
- Bảng người xem không có trần/TTL; mỗi state/event sắp xếp toàn bộ bảng.
- Có **1.605 hồ sơ**, khoảng 795 KB dữ liệu người xem và URL avatar trong file
  tracked `data/merit-state.json`. Cần xem lại việc đưa dữ liệu phiên live lên Git.
- Đề xuất: flush định kỳ có trần thời gian, atomic write/SQLite, flush khi thoát,
  backup, giới hạn dữ liệu và tách dữ liệu chạy khỏi source control.

### R6 — Trung bình: cấu hình âm thanh giữa admin/overlay không đồng bộ đầy đủ

- Vị trí: `server.js:666–675`; `public/js/live.js:418–435,778–794`;
  `public/js/audio.js:66–67,296–415`.
- Frontend chỉ áp dụng musicVolume/ttsVolume từ state; không áp dụng
  ttsEnabled, ttsVoice, bgMusicEnabled, sfxEnabled, smokeDensity. Nút bật/tắt
  tiếng chỉ sửa engine tại tab đó, không đồng bộ sang OBS source khác.
- Mỗi tab live tự phát tiếng; mở cả preview và OBS có thể nghe chồng tiếng.
  Cần click unlock âm thanh riêng trong ngữ cảnh Browser Source.
- Ducking có sàn 0,08 nên musicVolume=0 vẫn có thể phát nhạc khi đọc TTS.
- Đề xuất: đồng bộ toàn bộ settings, xác định tab/source phát tiếng, giữ mức 0
  khi ducking, báo lỗi phát thay vì bỏ qua.

### R7 — Trung bình: TTS thiếu kiểm soát tải và tự phục hồi

- Vị trí: `server.js:145–239,1133–1140`;
  `tts_worker.py:74–80,142–198`; `public/js/audio.js:312–332`.
- Python chỉ xử lý tuần tự; backend không giới hạn pending, cache không có quota
  hay dọn dẹp. Mỗi tab prefetch dù hàng chờ UI chỉ giữ 6; bỏ item UI không hủy
  công việc đã gửi. Timeout 30 giây không hủy tổng hợp phía Python.
- Worker chết không tự khởi động lại. Không có fallback Web Speech trong code
  dù spec mô tả có. TTS cắt còn 160 ký tự, có thể mất phần cuối lời khấn/quẻ.
- `/api/tts` gọi `.trim()` trên input chưa kiểm tra kiểu trong async handler
  Express 4; đầu vào không phải string có thể tạo rejected promise ngoài luồng
  xử lý lỗi chuẩn. Đây là phát hiện tĩnh, chưa thử gây crash server thật.
- Đề xuất: queue có trần theo server, rate limit, timeout/cancel, worker restart
  có backoff, cache quota, báo trạng thái rõ cho người vận hành.

### R8 — Trung bình: vòng đời kết nối TikTok còn yếu

- Vị trí: `server.js:833–995,1215–1220`; `public/js/live.js:395–409,621–637`.
- Tự kết nối kênh hardcode `zenfacts_vn` lúc khởi động. Nhiều live.connect song
  song dùng chung biến tiktokConn; callback kết nối cũ vẫn sửa state hiện tại.
  Không thấy cơ chế reconnect ở code ứng dụng; khi rớt mạng cần giám sát.
- Quick dock gửi RPC qua WS nhưng bỏ qua response lỗi; không hiển thị lastError.
  Frontend bỏ qua notification temple.viewerCount/tts.status, nên số người xem
  và trạng thái TTS có thể không cập nhật ngay.
- Connector upstream tự mô tả là unofficial/reverse engineered; tính ổn định
  phụ thuộc TikTok và dịch vụ ký URL. Chưa xác nhận khả năng kết nối trên máy này.
- Đề xuất: opt-in kết nối, một state machine theo connection ID, serialize
  connect/disconnect, reconnect có backoff, hiển thị lỗi rõ.

### R9 — Thấp: README/spec và trải nghiệm chưa khớp hoàn toàn

- `public/assets/bg_chant.m4a` không được commit và bị ignore: nhạc tụng niệm
  không có trong clone mới, engine dùng giai điệu tổng hợp dự phòng.
- Bộ quẻ thực tế có 10; comment nói 18, spec nói 24. Nội dung quẻ có lời khẳng
  định tài lộc/sức khỏe; cần duyệt nội dung trước phát công khai. Chưa đánh giá
  chính sách nền tảng hay quyền sử dụng tài sản/nhạc cho hoạt động thương mại.
- `includes('que')` bắt cả `que toi o Hue` thành xin xăm; nên nhận từ/cụm từ rõ.
- Không có bộ điều phối hàng chờ nghi lễ: sự kiện đến dồn có thể ghi đè người
  hành lễ và timer hình ảnh, dù hàng chờ TTS có giới hạn riêng.
- JSON-RPC chỉ hỗ trợ một phần: không batch, notification vẫn có HTTP response,
  không validate version, lỗi method/params dùng chung -32000.
- Các tuyên bố 0,1% CPU/treo 24/7 trong README chưa được benchmark trên máy này.
- Python dependencies không có lock/version pin; model cũng tải từ upstream
  main. Môi trường mới có thể khác với máy tác giả dù JavaScript có lockfile.

## Kiểm chứng đã làm trước khi chạy ứng dụng

- `npm ci --ignore-scripts --no-audit --no-fund`: 118 package cài được từ lock.
- `npm audit --json`: 0 advisory (info/low/moderate/high/critical đều 0), tại
  thời điểm kiểm tra; không phải bằng chứng code ứng dụng an toàn.
- `node --check` với server và ba script frontend: qua; Python AST parse: qua.
- Harness VM ngoài repo, không mở network/worker hay ghi dữ liệu thật: bảy
  handler trả đúng loại event và điểm cơ bản; tái hiện R2, R3, R4 và parser R9.
- Đối chiếu code connector đã khóa ở 2.5.0 để tránh suy luận sai event social.
- Smoke test trên server Bun thật: bảy nghi thức đều HTTP 200 và có event;
  WebSocket nhận đủ bảy broadcast và trả lời `temple.getState`.
- `/`, `/admin`, `/?obs=1` đều HTTP 200; nhạc tụng niệm 404, đúng với R9.
- Microsoft Edge, phiên QA riêng: bật âm thanh, dâng hương, hoa đăng, xin xăm,
  giả lập bình luận từ admin đều cập nhật tên/điểm/hình ảnh; quẻ hiện được.
  Khung OBS sạch 9:16 không hiện quick dock. Chưa kiểm thử trình duyệt nhúng OBS.
- TTS ready với tám voice. Câu thử Mai Chi tạo WAV 48.000 Hz, dài 2,72 giây,
  tổng hợp trong 1,37 giây; waveform hữu hạn, peak 0,6729. Đây là kiểm tra tạo
  file và luồng UI, không phải đánh giá chất lượng giọng bằng người nghe.
- `lsof` xác nhận chỉ listen `127.0.0.1:3000`; TikTok connected=false,
  connecting=false. Checkout gốc sạch sau toàn bộ thao tác thử.
- Một mẫu khi nghỉ sau chạy: Bun khoảng 59 MiB RSS, worker khoảng 1,39 GiB RSS;
  CPU mỗi tiến trình 0,0% trong mẫu đó. Chưa đo tải đông người hay phiên dài.

### Lỗi môi trường TTS và cách đã khắc phục tại máy

Đã cài Python 3.12.11, zerotts 0.1.5, onnxruntime 1.31.1, huggingface-hub 2.4.0.
Model snapshot `18d6681518a4becaed3c125ceb3c7db59905fe86` gồm 38 file,
1.047.985.886 byte. Lần đầu worker thoát: ORT báo `External data path escapes
model directory`; file ONNX và file `.data` trong cache Hugging Face là symlink
đến các thư mục blob khác nhau. Worker không tự phục hồi, đúng với R7.

Khắc phục môi trường: copy snapshot thành file thường, giữ nguyên cấu trúc trong
`.venv/models/zerotts`, rồi đặt `ZEROTTS_MODEL_PATH` trỏ thư mục đó khi khởi động.
Không hạ phiên bản hay tắt kiểm tra an toàn của ONNX Runtime; không sửa source
ứng dụng hoặc package đã cài. Worker khởi tạo lại thành công trong 6,35 giây.
Cache/model/environment đều nằm ngoài commit; clone mới cần chuẩn bị lại.

## Thử local trước khi cài OBS

Chỉ mở server trên loopback; không tự kết nối tài khoản mặc định. Dùng
`http://127.0.0.1:3000/` để bấm hiệu ứng, bật âm thanh, thử đọc lời khấn và giả
lập bình luận qua `/admin`. Chế độ `/?obs=1` là khung sạch dùng sau này.

Trong worktree thử hiện tại đã có launcher local bị ignore tại
`.venv/temple-local-only.cjs`. Launcher chỉ sửa runtime để bind loopback và đặt
`TIKTOK_USER` thành khoảng trắng: resolver từ chối trước khi gọi mạng. Vì code
hiện chưa có cờ tắt auto-connect, log sẽ có cảnh báo username trống dự kiến.
Sau khi dừng tiến trình cũ, có thể chạy lại **từ thư mục worktree** bằng:

```sh
ZEROTTS_MODEL_PATH="$PWD/.venv/models/zerotts" \
  bun --preload ./.venv/temple-local-only.cjs server.js
```

Launcher/model là chuẩn bị riêng cho lần thử trên máy, không phải bản vá bảo mật
cho lệnh `bun start`. Không dùng `bun start` để suy ra có cùng giới hạn mạng.

Khi đã cài OBS: thêm Browser Source 1080 × 1920, URL
`http://127.0.0.1:3000/?obs=1`; bật phát/capture audio và click unlock trong
Interact. Bắt đầu live trong công cụ phát rồi nhập đúng @username để kết nối.
Chưa nên dùng quà thật để đánh giá tính đúng của combo trước khi sửa R4.

## Nguồn đối chiếu

- [TikTok Live Connector upstream](https://github.com/zerodytrash/TikTok-Live-Connector)
  và source package 2.5.0 đã cài từ lock.
- [ZeroTTS README chính thức](https://github.com/zeroweight-ai/ZeroTTS/blob/main/README.md):
  hướng dẫn cài, voice Mai Chi, tải weights khoảng 900 MB lần đầu.

Báo cáo này chưa sửa logic ứng dụng. Ưu tiên sửa R1–R4 trước, sau đó persistence,
cấu hình âm thanh, giới hạn TTS và khả năng phục hồi kết nối.

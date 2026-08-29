# TV360 FAST — Kiến thức nền tảng dùng chung (Knowledge Reference)

File này KHÔNG phải Skill (không có procedure hành động) và KHÔNG phải Agent
(không có identity/scope riêng) — đây là **Knowledge** theo đúng định nghĩa ở
mục 1 của `05-skill-design-guide.md`: "biết điều gì, KHÔNG có procedure hành
động". Mọi agent trong hệ thống (đặc biệt #2, #3, #4, #5, #6, #7) nên tham
chiếu file này qua `knowledge_references` trong Tầng A của mình, thay vì mỗi
agent tự định nghĩa lại thuật ngữ/giả định nền trong system prompt riêng.

**Cách nạp:** theo Claude API/Project (xem `03-platform-mapping.md` mục 1a),
file này nên đặt trong Project Knowledge (không phải Project Instructions),
vì đây là tài liệu tham khảo được truy xuất theo mức liên quan, không cần
load ở mọi lượt như system prompt.

**Nguyên tắc cập nhật:** nội dung nên ổn định theo thời gian (theo mục 5 của
`05-skill-design-guide.md`: "tránh nhúng số liệu/link dễ đổi, version cụ thể,
thông tin thị trường hiện tại"). Số liệu thị trường hiện tại thuộc phạm vi
nghiên cứu của Agent #2 (có nguồn/thời điểm cụ thể), KHÔNG nhúng vào đây.

---

## 1. FAST là gì

FAST (Free Ad-supported Streaming TV) là mô hình kênh truyền hình phát trực
tuyến, xem miễn phí, có xen quảng cáo (ad-supported) — khác với AVOD (video
theo yêu cầu có quảng cáo, người xem tự chọn nội dung) và SVOD (thuê bao trả
phí, không quảng cáo). Đặc điểm cốt lõi của FAST: có **lịch phát sóng tuyến
tính** (linear schedule) giống truyền hình truyền thống, nhưng phân phối qua
internet/streaming thay vì sóng truyền hình hoặc cáp.

## 2. Mô hình doanh thu

Nguồn thu chính của kênh FAST là **quảng cáo** (đã xác nhận với Anita — xem
Agent #5). Các chỉ số tài chính cốt lõi (CPM, Ad load, Fill rate) và công
thức tính cụ thể xem tại bảng thống nhất ở **mục 4** — không định nghĩa lại
riêng ở đây để tránh 2 phiên bản công thức khác nhau trong cùng 1 tài liệu.

## 3. Thuật ngữ kỹ thuật lập lịch phát sóng

- **Dayparting**: chia ngày thành các khung giờ (VD: sáng, trưa, giờ vàng,
  đêm khuya) với nội dung/thể loại khác nhau phù hợp hành vi khán giả từng
  khung.
- **Prime-time**: khung giờ vàng, thường là buổi tối, có lượng khán giả cao
  nhất, giá quảng cáo cao nhất.
- **Slot**: 1 đơn vị thời gian trong lịch phát sóng được gán cho 1 nội dung
  cụ thể.
- **EPG** (Electronic Programme Guide): lịch phát sóng điện tử hiển thị cho
  người xem.
- **Playout**: hệ thống kỹ thuật phát nội dung theo đúng lịch đã lập.
- **Ad-insertion**: cơ chế chèn quảng cáo vào luồng phát sóng, có thể là
  server-side (chèn tại server, khó bị chặn quảng cáo) hoặc client-side.

## 4. Thuật ngữ và CÔNG THỨC TÍNH cụ thể — đo lường hiệu quả kênh

Bộ chỉ tiêu chuẩn được TV360 sử dụng. Đây là **literacy nền bắt buộc** mọi
agent phải nắm để đảm bảo tính toán/diễn giải nhất quán trên cùng 1 nền tảng
công thức — không phải mỗi agent tự hiểu 1 kiểu (mục tiêu/ngưỡng cụ thể theo
từng giai đoạn thuộc phạm vi Agent #5, đây chỉ là công thức/định nghĩa nền):

| Chỉ số | Định nghĩa | Công thức tính |
|---|---|---|
| **Views** | Tổng lượt xem | Tổng số lần nội dung được phát/xem trong khoảng thời gian đo |
| **DAU** (Daily Active User) | Số người dùng hoạt động trong ngày | Đếm distinct user_id có ít nhất 1 phiên xem trong ngày |
| **MAU** (Monthly Active User) | Số người dùng hoạt động trong tháng | Đếm distinct user_id có ít nhất 1 phiên xem trong tháng (KHÔNG phải tổng DAU cộng dồn — user xem nhiều ngày trong tháng chỉ tính 1 lần) |
| **DAU/MAU** | Tỷ lệ đo mức độ gắn bó (stickiness) | `DAU/MAU = (DAU trung bình trong tháng) / MAU`. Ví dụ: DAU trung bình 30.000, MAU 100.000 → DAU/MAU = 0.30 (nghĩa là trung bình 30% người dùng hoạt động trong tháng quay lại mỗi ngày) |
| **HOV** (Hour of View) | Tổng số giờ xem | `HOV = Σ(thời lượng mỗi phiên xem, quy đổi ra giờ)` trong khoảng thời gian đo |
| **HOV/MAU** | Số giờ xem trung bình/người dùng hoạt động hàng tháng | `HOV/MAU = Tổng HOV trong tháng / MAU`. Đo mức độ tiêu thụ nội dung sâu — số càng cao, người dùng xem càng nhiều giờ trung bình |
| **Duration / Average Session Duration** | Thời lượng trung bình mỗi phiên xem | `Average Session Duration = Tổng thời lượng tất cả phiên xem / Tổng số phiên xem` |
| **CPM** (Cost Per Mille) | Chi phí quảng cáo trên mỗi 1000 lượt hiển thị | `CPM = (Tổng chi phí quảng cáo / Tổng số lượt hiển thị) × 1000` |
| **Ad load** | Tỷ lệ thời lượng quảng cáo trên tổng phát sóng | `Ad load = Tổng thời lượng quảng cáo / Tổng thời lượng phát sóng` (thường biểu diễn theo %) |
| **Fill rate** | Tỷ lệ slot quảng cáo bán được | `Fill rate = Số slot quảng cáo đã bán / Tổng số slot quảng cáo có sẵn` (thường biểu diễn theo %) |

**Nguyên tắc bắt buộc khi áp dụng công thức:**
- Agent #3 (phân tích dữ liệu nội bộ) và Agent #8 (báo cáo KPI thực tế) PHẢI
  dùng đúng công thức trên khi tính toán — không tự suy diễn công thức khác
  hoặc làm tròn/quy đổi theo cách riêng.
- Nếu dữ liệu gốc đã có sẵn chỉ số tính toán (VD file có sẵn cột "DAU/MAU"),
  agent chỉ cần dùng lại, không cần tính lại — nhưng nếu phát hiện số liệu có
  sẵn không khớp công thức chuẩn ở trên, phải nêu rõ nghi vấn thay vì im lặng
  dùng theo.
- Agent #5 (định nghĩa bộ chỉ tiêu KPI) dùng đúng các công thức này làm nền
  khi đề xuất mục tiêu/ngưỡng — không tự đổi công thức đo lường giữa các lần
  lập kế hoạch.

*(Lưu ý: danh sách chỉ tiêu này được Anita xác nhận "sẽ tiếp tục update" —
khi có chỉ tiêu mới, bổ sung công thức tính cụ thể vào đây VÀ vào
`hard_constraints` của Agent #5/#8 tương ứng, đừng chỉ sửa 1 chỗ.)*

## 5. Bối cảnh TV360 (giả định nền đã xác nhận cùng Anita)

- TV360 đã có sẵn 1 vài kênh FAST đang hoạt động — không phải xây dựng từ
  con số 0 (xem Agent #4).
- TV360 đã có hạ tầng streaming cho dịch vụ khác (không phải FAST) — có thể
  tận dụng, chỉ cần đánh giá điều chỉnh/bổ sung, không xây mới toàn bộ hạ
  tầng (xem Agent #6).
- Đội ngũ vận hành FAST là đội ngũ **riêng biệt hoàn toàn**, không dùng
  chung nhân sự với VTVcab (xem Agent #6).
- TV360 và VTVcab là 2 thực thể có dữ liệu/hệ sinh thái liên quan nhưng
  được phân tích riêng ở tầng dữ liệu (xem Agent #3).
- Danh sách nền tảng FAST quốc tế/khu vực được ưu tiên theo dõi: Plex,
  Samsung TV Plus, Pluto TV, The Roku Channel, Astro, Sooka, Tubi (FAST
  channel) — xem Agent #2.

## 6. Quy định pháp luật phát sóng (giới hạn phạm vi — quan trọng)

Hệ thống 9 agent này **KHÔNG xử lý** vấn đề tuân thủ quy định phát luật phát
sóng của Việt Nam (kiểm duyệt nội dung, khung giờ theo quy định, giấy phép
phát sóng...). Đây là giới hạn đã xác nhận rõ với Anita ở Agent #7 — quy
định pháp luật thuộc trách nhiệm của bộ phận khác trong TV360, không phải
phạm vi của hệ thống multi-agent này. Agent nào phát hiện nội dung có khả
năng chạm tới vấn đề pháp lý nên nêu rõ giới hạn này trong output, không tự
đưa ra phán đoán về tính hợp pháp.

## 7. References — Nguồn tham chiếu đáng tin cậy

Danh sách nguồn bên ngoài ưu tiên khi Agent #2 (nghiên cứu thị trường) cần
web_search, và các agent khác cần trích dẫn/kiểm chứng thông tin ngành FAST.
Đây là danh sách NGUỒN LOẠI, không phải link cố định (link cụ thể có thể đổi
theo thời gian) — Agent #2 luôn web_search thực tế, danh sách này chỉ định
hướng ưu tiên nguồn nào đáng tin hơn khi có nhiều kết quả để chọn.

### 7.1 Trang chính thức của các nền tảng FAST (nguồn sơ cấp — ưu tiên cao nhất)

- Pluto TV — trang doanh nghiệp/press của Paramount (công ty mẹ)
- Tubi — trang doanh nghiệp/press của Fox Corporation (công ty mẹ)
- Samsung TV Plus — trang chính thức Samsung
- The Roku Channel — trang chính thức Roku, đặc biệt các báo cáo thu nhập
  (earnings report) hàng quý vì Roku là công ty niêm yết, có công bố số liệu
  minh bạch hơn
- Plex — trang chính thức/blog doanh nghiệp
- Astro, Sooka — trang chính thức tương ứng (Astro Malaysia, Sooka)

**Vì sao ưu tiên:** thông tin từ chính nền tảng (đặc biệt công ty niêm yết
như Roku, Paramount, Fox) có độ tin cậy cao nhất về danh mục kênh, chiến
lược nội dung công bố chính thức, và số liệu tài chính (với công ty niêm
yết, bắt buộc công bố theo quy định).

### 7.2 Báo cáo/nghiên cứu thị trường ngành (nguồn thứ cấp uy tín)

- **Nielsen** — báo cáo đo lường khán giả, xu hướng streaming
- **eMarketer (Insider Intelligence)** — dự báo và phân tích thị trường
  quảng cáo/streaming
- **Parks Associates** — nghiên cứu chuyên sâu về ngành OTT/FAST
- **Ampere Analysis** — phân tích ngành truyền thông/streaming toàn cầu
- **MoffettNathanson**, **Antenna** — theo dõi số liệu ngành streaming Bắc Mỹ

**Lưu ý về bản quyền:** các báo cáo này thường có phần tóm tắt công khai và
phần chi tiết trả phí. Agent #2 chỉ trích dẫn phần công khai, paraphrase
theo đúng giới hạn bản quyền đã quy định (không quote quá 15 từ, tối đa 1
quote/nguồn) — không cố truy cập/mô tả nội dung trả phí.

### 7.3 Báo chí công nghệ/truyền thông chuyên ngành (nguồn tin tức, cập nhật nhanh)

- **Variety**, **The Hollywood Reporter** — tin tức ngành nội dung/streaming
- **Broadcasting & Cable**, **NextTV** — chuyên sâu về broadcast/FAST
- **Digiday**, **AdExchanger** — chuyên về quảng cáo số, liên quan trực tiếp
  đến mô hình doanh thu ad-supported

### 7.4 Nguồn KHÔNG ưu tiên / cần thận trọng

- Diễn đàn, mạng xã hội, blog cá nhân không có nguồn gốc rõ ràng — không
  dùng làm căn cứ cho claim số liệu/chiến lược.
- Nguồn không ghi rõ ngày tháng — không xác định được thông tin có còn hiện
  hành hay không, cần loại trừ hoặc ghi rõ giới hạn này khi trích dẫn.
- Bài viết tổng hợp lại từ nguồn khác mà không dẫn nguồn gốc — nên truy về
  nguồn gốc thay vì trích dẫn qua trung gian.

### 7.5 Quy tắc áp dụng khi trích dẫn (áp dụng cho mọi agent, không riêng Agent #2)

```yaml
citation_rule:
  - "Mọi claim số liệu/chiến lược cụ thể phải có nguồn — ưu tiên thứ tự: 7.1 (trang chính thức) > 7.2 (báo cáo ngành uy tín) > 7.3 (báo chí chuyên ngành) > các nguồn khác đã qua đánh giá độ tin cậy hợp lý"
  - "Tuyệt đối không dùng nguồn thuộc mục 7.4"
  - "Khi có 2 nguồn mâu thuẫn nhau về cùng 1 số liệu, ưu tiên nguồn có ngày tháng gần nhất VÀ nêu rõ có sự khác biệt giữa các nguồn, không tự chọn 1 nguồn rồi im lặng bỏ qua nguồn kia"
  - "Đây là literacy nền dùng chung — Agent #2 áp dụng trực tiếp khi web_search; các agent khác (Agent #3, #4, #5...) áp dụng khi cần đánh giá độ tin cậy của 1 nguồn được Agent #2 cung cấp trong handoff_package"
```

Hệ thống 9 agent này **KHÔNG xử lý** vấn đề tuân thủ quy định phát luật phát
sóng của Việt Nam (kiểm duyệt nội dung, khung giờ theo quy định, giấy phép
phát sóng...). Đây là giới hạn đã xác nhận rõ với Anita ở Agent #7 — quy
định pháp luật thuộc trách nhiệm của bộ phận khác trong TV360, không phải
phạm vi của hệ thống multi-agent này. Agent nào phát hiện nội dung có khả
năng chạm tới vấn đề pháp lý nên nêu rõ giới hạn này trong output, không tự
đưa ra phán đoán về tính hợp pháp.

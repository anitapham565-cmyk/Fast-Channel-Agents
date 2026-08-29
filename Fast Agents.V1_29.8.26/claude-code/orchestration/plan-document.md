# PLAN.md — Kim chỉ nam quy trình cố định (Master Playbook)

**Sửa đổi quan trọng so với bản trước:** PLAN.md KHÔNG còn là output động
được trích xuất từ Agent #4 sau mỗi request. PLAN.md giờ là **1 tài liệu
TĨNH, viết 1 lần, dùng chung cho mọi request** — đóng vai trò kim chỉ nam
(playbook) mà Agent #1 đọc TRƯỚC khi bắt đầu điều phối bất kỳ workflow nào,
để biết chuẩn các bước cần bám theo nhằm xây dựng báo cáo chiến lược kênh
FAST cũng như vận hành, đánh giá sau này.

**Khác với Plan Tracking:** `plan-tracking.md` vẫn là trạng thái ĐỘNG (mỗi
request có 1 bản plan_tracking riêng, cập nhật theo thời gian thực). PLAN.md
(tài liệu này) là chuẩn TĨNH mà plan_tracking phải tuân theo cấu trúc. Nói
cách khác: PLAN.md là "bản thiết kế", plan_tracking là "nhật ký thi công
theo đúng bản thiết kế đó cho 1 công trình cụ thể".

**Agent #4 không còn liên quan tới việc tạo PLAN.md.** Phần "Kế hoạch triển
khai" trong output của Agent #4 (milestone/giai đoạn triển khai kênh FAST
NGOÀI ĐỜI THỰC — khác hoàn toàn với PLAN.md ở đây) vẫn giữ nguyên như thiết
kế gốc, không đổi.

---

## 1. Vị trí trong hệ thống

```
Trước khi xử lý BẤT KỲ request nào từ Anita:
        │
        ▼
Agent #1 đọc PLAN.md (tài liệu này) — không đổi theo từng request
        │
        ▼
Agent #1 dùng PLAN.md làm chuẩn để:
  - Biết đúng thứ tự gọi agent nào trong pha Strategy (#2→#3→#4→#5+#6→#7→#8→#9)
  - Biết đúng agent nào được gọi trong pha Operations (#7 và/hoặc #8)
  - Biết quy tắc song song, quy tắc retry, quy tắc dừng
  - Đối chiếu xem workflow hiện tại có đang lệch khỏi playbook chuẩn không
```

PLAN.md tách biệt khỏi luồng 9 agent theo nghĩa: **không agent con nào (#2
đến #9) đọc hoặc tham chiếu trực tiếp PLAN.md** — chỉ Agent #1 đọc, vì đây
là tài liệu điều phối cấp hệ thống, không phải kiến thức nghiệp vụ.

## 2. Nội dung PLAN.md — Playbook chuẩn

### 2.1 Playbook cho pha Strategy (xây báo cáo chiến lược)

```yaml
strategy_playbook:
  trigger: "Yêu cầu có từ khoá 'chiến lược', 'báo cáo tổng thể', 'quy hoạch kênh mới'"
  steps:
    - order: 1
      agents: ["agent_2", "agent_3"]
      execution: "song song"
      purpose: "Thu thập insight thị trường (agent_2) và dữ liệu nội bộ (agent_3) độc lập với nhau"
    - order: 2
      agents: ["agent_4"]
      execution: "tuần tự, chờ bước 1 xong"
      purpose: "Tổng hợp thành đề xuất chiến lược, dựa trên output bước 1"
    - order: 3
      agents: ["agent_5", "agent_6"]
      execution: "song song, chờ bước 2 xong"
      purpose: "Đánh giá tài chính (agent_5) và điều kiện vận hành (agent_6) song song, cùng dùng output agent_4"
    - order: 4
      agents: ["agent_7"]
      execution: "tuần tự, chờ bước 2 xong (không cần chờ bước 3)"
      purpose: "Xây khung phát sóng tổng thể dựa trên định hướng của agent_4"
    - order: 5
      agents: ["agent_8"]
      execution: "tuần tự, chờ bước 3 xong (cần bộ chỉ tiêu từ agent_5)"
      purpose: "Xác lập KPI baseline dự kiến, dựa trên bộ chỉ tiêu agent_5 định nghĩa"
    - order: 6
      agents: ["agent_9"]
      execution: "tuần tự, chờ TẤT CẢ bước 1-5 xong"
      purpose: "Đánh giá per-agent-section, duyệt/từ chối từng phần"
  retry_rule: "Nếu agent_9 trả về có phần rejected, route lại ĐỒNG THỜI mọi agent có phần bị từ chối, tăng retry_count chung lên 1, tối đa 3 vòng"
  stop_rule: "Dừng khi TẤT CẢ phần approved, HOẶC retry_count = 3 mà vẫn còn phần rejected"
```

### 2.2 Playbook cho pha Operations (vận hành liên tục)

```yaml
operations_playbook:
  trigger: "Yêu cầu có từ khoá 'lịch hôm nay', 'KPI tuần/tháng', 'cập nhật vận hành'"
  steps:
    - order: 1
      agents: ["agent_7"]
      condition: "khi yêu cầu liên quan lập lịch phát sóng"
      purpose: "Lập lịch chi tiết hàng ngày, dựa trên khung tổng thể đã có (từ lần Strategy trước) + danh sách content khả dụng"
    - order: 2
      agents: ["agent_8"]
      condition: "khi yêu cầu liên quan báo cáo KPI"
      purpose: "Thu thập/đánh giá KPI định kỳ, dựa trên bộ chỉ tiêu đã có (từ lần Strategy trước)"
  approval_rule: "KHÔNG gọi agent_9 trong pha này — không cần phê duyệt"
  note: "2 bước trên độc lập, có thể chỉ chạy 1 trong 2 tuỳ yêu cầu cụ thể, không bắt buộc chạy cả 2"
```

### 2.3 Ràng buộc chung áp dụng cho cả 2 pha

```yaml
global_constraints:
  max_parallel: 2
  max_retries: 3    # chỉ áp dụng pha Strategy, tính chung cho cả báo cáo
  hub_and_spoke: true    # agent con không tự gọi agent khác, chỉ Agent #1 điều phối
  handoff_format: "Mọi agent con dùng Skill 'write-handoff-summary' (trừ agent_9 dùng 'format-partial-approval')"
  knowledge_baseline: "Mọi agent phải tham chiếu tv360-fast-glossary.md để đảm bảo dùng chung 1 nền kiến thức, 1 ngôn ngữ thuật ngữ (xem chi tiết lý do ở README mục 4.2 sau khi cập nhật)"
```

## 3. Cách Agent #1 dùng PLAN.md trong thực tế

```yaml
usage_rule:
  - "Khi bắt đầu xử lý 1 request mới, Agent #1 đọc PLAN.md (qua tool Read) để xác nhận đúng playbook áp dụng theo pha (Strategy/Operations)"
  - "Agent #1 KHÔNG tự sáng tạo thứ tự gọi agent khác với playbook đã định nghĩa — nếu tình huống thực tế cần lệch khỏi playbook (VD: 1 agent lỗi liên tục), phải dừng và báo Anita, không tự ý đổi thứ tự"
  - "PLAN.md là tài liệu TĨNH — Agent #1 không được tự ý ghi đè/sửa nội dung PLAN.md trong lúc vận hành. Nếu cần thay đổi playbook (VD: đổi thứ tự agent, thêm agent mới), đây là thay đổi THIẾT KẾ, cần Anita xác nhận và cập nhật thủ công file này, không phải việc Agent #1 tự động hoá"
  - "plan_tracking (trạng thái động, xem plan-tracking.md) phải luôn đối chiếu được với đúng cấu trúc steps trong PLAN.md — mỗi entry trong plan_tracking.strategy_steps tương ứng 1:1 với 1 step trong strategy_playbook.steps"
```

## 4. Vì sao tách PLAN.md khỏi luồng 9 agent

- **Ổn định:** vì đây là playbook cố định (không đổi theo request), tách riêng
  giúp không phải nhúng lại toàn bộ logic điều phối vào system prompt của Agent
  #1 mỗi lần — đúng nguyên tắc "Lazy-load knowledge" (`04-token-efficiency.md`
  mục 2): Agent #1 chỉ cần đọc khi cần đối chiếu, không phải lúc nào cũng giữ
  toàn văn trong context.
- **Version hoá dễ dàng:** khi quy trình cần điều chỉnh (VD: thêm Agent #10
  trong tương lai), chỉ sửa 1 file PLAN.md, không phải sửa rải rác trong system
  prompt của Agent #1 lẫn nội dung SOP.
- **Không lẫn với nội dung nghiệp vụ:** PLAN.md là quy trình kỹ thuật (ai chạy
  khi nào), khác hoàn toàn với "Kế hoạch triển khai kênh FAST" (nội dung kinh
  doanh) mà Agent #4 tạo ra — giữ 2 khái niệm này tách bạch tránh nhầm lẫn như
  đã xảy ra ở lần thiết kế trước.

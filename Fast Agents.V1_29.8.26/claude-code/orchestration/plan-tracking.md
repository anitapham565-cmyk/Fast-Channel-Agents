# PLAN TRACKING — Cơ chế theo dõi tiến độ độc lập

Đây KHÔNG phải nội dung chiến lược (đó là output của Agent #4 — kế hoạch
triển khai theo milestone). Đây là 1 **cấu trúc trạng thái kỹ thuật** riêng
biệt, giống 1 todo-list/tiến độ workflow, được Agent #1 cập nhật mỗi khi 1
agent con hoàn thành — phục vụ mục đích: Anita luôn biết workflow đang ở
đâu, và nếu phiên làm việc bị gián đoạn, có thể xem lại tiến độ mà không cần
Agent #1 chạy lại từ đầu.

Vì schema gốc (`00-generic-schema.md`) không có tầng riêng cho việc này, đây
là 1 bổ sung áp dụng cụ thể cho project TV360 FAST, gắn vào Tầng B + F của
Agent #1 (không phải 1 agent/skill mới).

---

## 1. Cấu trúc dữ liệu Plan Tracking

```yaml
plan_tracking:
  request_id: ""              # ID định danh cho 1 lần chạy workflow (Strategy hoặc Operations)
  phase: "strategy | operations"
  started_at: ""
  updated_at: ""

  # Chỉ có ở pha Strategy — theo dõi từng agent trong chuỗi #2-#9
  strategy_steps:
    - agent: "agent_2"
      status: "pending | in_progress | completed | failed"
      started_at: ""
      completed_at: ""
    - agent: "agent_3"
      status: "pending | in_progress | completed | failed"
      started_at: ""
      completed_at: ""
    - agent: "agent_4"
      status: "pending"
      depends_on: ["agent_2", "agent_3"]
    - agent: "agent_5"
      status: "pending"
      depends_on: ["agent_4"]
    - agent: "agent_6"
      status: "pending"
      depends_on: ["agent_4"]
    - agent: "agent_7"
      status: "pending"
      depends_on: ["agent_4"]
    - agent: "agent_8"
      status: "pending"
      depends_on: ["agent_7"]
    - agent: "agent_9"
      status: "pending"
      depends_on: ["agent_5", "agent_6", "agent_7", "agent_8"]

  approval_rounds:                # Chỉ áp dụng pha Strategy, cập nhật mỗi vòng duyệt
    - round: 1
      submitted_at: ""
      result:
        agent_2: "approved"
        agent_3: "approved"
        agent_4: "approved"
        agent_5: "rejected"
        agent_6: "approved"
        agent_7: "approved"
        agent_8: "approved"
      overall_status: "partial_approved"
    # round 2, 3... thêm khi có retry

  retry_count: 0                  # Bộ đếm CHUNG, tối đa 3 (khớp Tầng F Agent #1)

  # Chỉ có ở pha Operations — không có chuỗi phụ thuộc phức tạp như Strategy
  operations_log:
    - task: "lập lịch phát sóng ngày X"
      agent: "agent_7"
      status: "completed"
      executed_at: ""
    - task: "báo cáo KPI tuần Y"
      agent: "agent_8"
      status: "completed"
      executed_at: ""

  final_status: "in_progress | completed | stopped_manual_review"
  stop_reason: ""                 # Chỉ điền nếu final_status = stopped_manual_review
```

## 2. Quy tắc cập nhật (gắn vào Tầng B của Agent #1)

Bổ sung vào SOP của Agent #1 (`agent-01-orchestrator.md`), thêm ngay sau mỗi
lần gọi agent con:

```yaml
plan_tracking_rule:
  - "Trước khi gọi bất kỳ agent con nào, cập nhật status của agent đó thành 'in_progress' trong plan_tracking, ghi started_at"
  - "Ngay sau khi agent con trả kết quả (qua handoff_contract), cập nhật status thành 'completed' hoặc 'failed', ghi completed_at"
  - "Sau mỗi lần Agent #9 trả kết quả duyệt, thêm 1 entry mới vào approval_rounds với đầy đủ result theo từng agent nguồn"
  - "Khi retry, tăng retry_count và chỉ đặt lại status của agent bị rejected về 'pending' rồi 'in_progress' — KHÔNG reset status của agent đã 'completed'/approved"
  - "Khi workflow dừng (do đạt max_retries hoặc do lỗi agent), cập nhật final_status và stop_reason rõ ràng"
  - "plan_tracking PHẢI phản ánh đúng trạng thái thực tế tại mọi thời điểm — không được đánh dấu 'completed' trước khi thực sự nhận được handoff_contract từ agent con"
```

## 3. Nơi lưu trữ

Theo xác nhận của Anita (cả 2 cơ chế): plan_tracking được:
1. **Truyền trực tiếp trong luồng** — Agent #1 giữ plan_tracking trong context
   của chính nó qua các lượt gọi API (đây là cơ chế bắt buộc, vì Agent #1 cần
   đọc lại để quyết định bước tiếp theo).
2. **Ghi vào storage chung** — mỗi lần cập nhật, Agent #1 (qua code driver bên
   ngoài, không phải chính Claude) ghi 1 bản snapshot plan_tracking vào nơi
   lưu trữ ngoài (VD: file JSON, database nhỏ, hoặc — nếu triển khai dưới dạng
   artifact có `window.storage` — dùng key dạng `plan_tracking:<request_id>`).
   Mục đích: Anita có thể tra cứu lại tiến độ 1 request cũ, hoặc nếu phiên
   làm việc bị ngắt giữa chừng, driver có thể đọc lại state từ storage thay vì
   mất toàn bộ tiến độ.

**Lưu ý kỹ thuật quan trọng:** việc ghi vào storage chung là trách nhiệm của
**code driver bên ngoài** (theo `03-platform-mapping.md` mục 0: Claude API
không có orchestration built-in), không phải bản thân Claude model tự ghi.
Agent #1 (model) chỉ có nhiệm vụ SINH RA nội dung plan_tracking đã cập nhật
trong output có cấu trúc của nó; code driver đọc output đó và ghi vào storage.

## 4. Plan tổng thể (khác với Plan Tracking)

Để tránh nhầm lẫn: **Plan tổng thể** (lộ trình chiến lược — giai đoạn, mục
tiêu kinh doanh, milestone triển khai) đã có sẵn, chính là phần "Kế hoạch
triển khai" trong output của **Agent #4** (xem `per_turn_output_structure`
của Agent #4 — mục "Kế hoạch triển khai" theo milestone/giai đoạn). Không
tạo thêm agent/skill mới cho việc này.

**Plan Tracking** (tài liệu này) là lớp kỹ thuật khác hoàn toàn: nó theo dõi
**tiến độ vận hành của chính hệ thống 9 agent** (agent nào đã chạy, agent nào
đang chờ, vòng duyệt thứ mấy) — không phải tiến độ triển khai kênh FAST thực
tế ngoài đời. Hai khái niệm này cần giữ tách biệt, đúng nguyên tắc ownership ở
mục 3 của `07-project-design-guide.md`.

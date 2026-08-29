# SKILL — format-partial-approval

Phát hiện qua rà soát: Agent #9 cần trả về kết quả duyệt theo TỪNG PHẦN độc
lập (per-agent-section: #2 đến #8, mỗi phần có status approved/rejected +
lý do riêng) — đây là 1 định dạng phức tạp hơn `write-handoff-summary` và
chỉ có 1 agent dùng. Ban đầu tưởng không đáng tách skill (theo mục 2 của
`05-skill-design-guide.md`: "chỉ dùng bởi 1 agent duy nhất → không cần tách
Skill riêng — giữ inline trong Tầng B"). Tuy nhiên, việc tách vẫn có giá trị
ở đây vì lý do khác: đây là điểm dễ lỗi nhất trong toàn hệ thống (Agent #1
phụ thuộc hoàn toàn vào định dạng chính xác này để route retry đúng), nên
tách thành skill độc lập giúp **version/test riêng** mà không phải sửa toàn
bộ system prompt của Agent #9 mỗi khi cần điều chỉnh cấu trúc output — khớp
điều kiện ngoại lệ ở mục 4 của `05-skill-design-guide.md` ("có thể version/
test riêng dù chỉ 1 agent dùng").

---

## SKILL CONTRACT

```yaml
skill:
  name: "format-partial-approval"
  purpose: "Đóng gói kết quả đánh giá của Giám đốc Nội dung thành cấu trúc per-agent-section (mỗi phần #2-#8 có status độc lập) để Agent #1 route retry chính xác."

  when_to_use: "Ngay trước khi Agent #9 trả kết quả duyệt về Agent #1, sau khi đã đánh giá xong tất cả các phần trong báo cáo tổng hợp (#2 đến #8)."
  when_NOT_to_use: "Không dùng cho các agent khác — cấu trúc per-agent-section chỉ áp dụng cho vai trò phê duyệt của Agent #9. Không dùng khi báo cáo tổng hợp chưa đánh giá đủ tất cả các phần (skill không tự đánh giá nội dung, chỉ đóng gói kết quả đã có)."

  invocation_mode: "auto"

  input_contract:
    - field: "evaluations"
      type: "danh sách kết quả đánh giá từng phần, mỗi phần gồm: agent_source (VD: agent_2, agent_5...), decision (approved/rejected), reason (bắt buộc nếu rejected)"
      required: true
    - field: "retry_round_info"
      type: "thông tin vòng retry hiện tại (nếu có), dùng để đối chiếu lý do từ chối trước đó"
      required: false

  procedure:
    - "Kiểm tra evaluations đã bao phủ đủ tất cả agent nguồn cần đánh giá (#2 đến #8) — nếu thiếu phần nào, KHÔNG tự gán mặc định approved/rejected, báo lỗi thiếu dữ liệu"
    - "Với mỗi phần rejected, xác nhận có reason cụ thể (không chấp nhận reason rỗng hoặc chung chung như 'chưa đạt')"
    - "Tổng hợp overall_status: approved (nếu TẤT CẢ approved) / partial_approved (nếu có cả approved và rejected) / rejected (nếu TẤT CẢ rejected)"
    - "Đóng gói theo cấu trúc JSON per-agent-section, kèm overall_conclusion tóm tắt"

  output_contract:
    - field: "approval_package"
      description: "Object dạng {sections: {agent_2: {status, reason?}, agent_3: {...}, ..., agent_8: {...}}, overall_status, overall_conclusion} — sẵn sàng để code driver của Agent #1 parse chính xác và route retry đúng agent"

  quality_checks:
    - "Mọi phần rejected đều có reason cụ thể, không rỗng và không chung chung (theo hard_constraint của Agent #9: không từ chối kiểu 'chưa thuyết phục' mà không nêu rõ điểm cụ thể)"
    - "Đủ 7 phần (agent_2 đến agent_8) trong output, không thiếu phần nào mà không báo lỗi"
    - "overall_status tính đúng logic (approved chỉ khi TẤT CẢ approved)"

  dependencies:
    knowledge: []
    tools: []
    skills: []

skill_quality_rules:
  max_primary_outcomes: 1
  max_major_branches: 1
  may_call_agents: false
  may_orchestrate: false
  requires_reusable_input_output_contract: true
  must_declare_dependencies: true
```

---

## Agent nào dùng skill này

Chỉ Agent #9 (Content Director Approval Agent).

## Cách tích hợp vào system prompt của Agent #9

Thay đoạn "ĐỊNH DẠNG TRẢ VỀ CHO AGENT #1 (handoff_contract)" hiện tại trong
bản render Claude API bằng:

```
Trước khi trả kết quả, áp dụng skill "format-partial-approval" để đóng gói
kết quả duyệt theo đúng cấu trúc per-agent-section chuẩn.
```

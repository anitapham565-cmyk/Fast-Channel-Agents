# SKILL — write-handoff-summary

Phát hiện qua rà soát: cấu trúc `handoff_contract` (status / conclusion /
evidence / confidence / next_action) bị lặp lại giống hệt ở cả 7 agent con
(#2 đến #8) trong hệ thống TV360 FAST — mỗi agent tự viết lại nguyên văn
định dạng này trong system prompt của mình thay vì tham chiếu 1 chuẩn chung.
Đây là dấu hiệu rõ của 1 Skill dùng chung (theo bảng quyết định mục 2 của
`05-skill-design-guide.md`: "capability tái dùng được ở ≥2 agent khác nhau").

---

## SKILL CONTRACT

```yaml
skill:
  name: "write-handoff-summary"
  purpose: "Đóng gói kết quả xử lý của 1 agent con thành định dạng handoff chuẩn (status/conclusion/evidence/confidence/attachment) để Agent #1 (Orchestrator) đọc và xử lý tiếp mà không cần parse tự do."

  when_to_use: "Ngay trước khi 1 agent con (#2 đến #8) trả kết quả về Agent #1, sau khi đã hoàn thành xử lý nghiệp vụ chính, tự kiểm tra (self_check_before_output), và tạo file đính kèm nếu agent đó có nhiệm vụ xuất file (.xlsx/.docx...)."
  when_NOT_to_use: "Không dùng cho Agent #1 (không cần đóng gói handoff cho chính nó) hoặc Agent #9 (dùng skill riêng do cấu trúc trả về khác — per-agent-section, không phải status đơn). Không dùng để thay thế self_check_before_output — skill này chỉ đóng gói, không kiểm tra chất lượng nội dung."

  invocation_mode: "auto"

  input_contract:
    - field: "raw_result"
      type: "nội dung kết quả nghiệp vụ đã hoàn thành (văn bản/dữ liệu có cấu trúc theo per_turn_output_structure của agent đó)"
      required: true
    - field: "status_hint"
      type: "string (success | partial | failed)"
      required: true
    - field: "confidence_hint"
      type: "string (Cao | Trung bình | Thấp)"
      required: false
    - field: "attachment_paths"
      type: "danh sách đường dẫn file đã tạo (VD .xlsx, .docx) — CHỈ áp dụng cho agent có nhiệm vụ xuất file (#3, #5, #7, #8); rỗng với agent không xuất file (#2, #4, #6)"
      required: false

  procedure:
    - "Trích xuất 1 câu tóm tắt ngắn gọn nhất (conclusion) từ raw_result — không diễn giải lại toàn bộ nội dung, chỉ nêu kết luận chính"
    - "Liệt kê evidence: các căn cứ/nguồn/số liệu cụ thể hỗ trợ conclusion, ở dạng ngắn gọn (không copy nguyên văn phần dài của raw_result)"
    - "Gán confidence dựa trên confidence_hint nếu có, hoặc suy ra từ độ đầy đủ của raw_result nếu không có hint rõ"
    - "Nếu attachment_paths có giá trị, xác nhận file thực sự tồn tại và đã qua bước verify theo đúng skill file-creation tương ứng (docx/xlsx/pptx) TRƯỚC KHI đưa đường dẫn vào handoff — không đính kèm file chưa được kiểm tra (chưa chạy recalc.py/validate.py)"
    - "Đóng gói theo cấu trúc chuẩn: status, conclusion, evidence, confidence, attachment (danh sách đường dẫn, có thể rỗng), next_action=N/A (vì agent con không tự quyết định bước tiếp theo)"

  output_contract:
    - field: "handoff_package"
      description: "Object có 5 field: status, conclusion, evidence, confidence, attachment (mảng đường dẫn file, rỗng nếu agent không xuất file) — sẵn sàng để Agent #1 đọc, route tiếp, và gộp file đính kèm vào bộ tài liệu cuối cùng trình Anita"

  quality_checks:
    - "conclusion không dài quá 2-3 câu — nếu dài hơn, đây là lỗi vi phạm mục đích 'tóm tắt', cần rút gọn thêm"
    - "evidence không copy nguyên văn toàn bộ raw_result — chỉ trích các điểm cốt lõi"
    - "Không tự thêm next_action cụ thể nếu agent gốc không có quyền quyết định bước tiếp theo (đúng theo scope_forbidden của từng agent)"
    - "Nếu agent có nhiệm vụ xuất file mà attachment rỗng, đây là lỗi — phải báo status=partial kèm lý do thiếu file, không im lặng bỏ qua"
    - "Nếu attachment có đường dẫn nhưng file chưa qua verify (recalc/validate), đây là lỗi nghiêm trọng — không đính kèm file lỗi công thức hoặc file corrupt"

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

Agent #2, #3, #4, #5, #6, #7, #8 — tất cả agent con trả kết quả trực tiếp về
Agent #1 theo cấu trúc `handoff_contract` chuẩn (status/conclusion/evidence/
confidence/next_action=N/A).

**Không áp dụng cho Agent #9** — Agent #9 trả về theo cấu trúc khác (per-agent-
section: nhiều status độc lập cho từng phần #2-#8), không khớp input_contract
của skill này. Agent #9 dùng skill riêng — xem `skill-format-partial-approval.md`.

## Cách tích hợp vào system prompt của agent con (rút gọn phần lặp lại)

Sau khi tách skill này, phần `handoff_contract` trong system prompt của mỗi
agent con (#2-#8) có thể rút gọn từ ~5 dòng mô tả chi tiết xuống còn 1 dòng
tham chiếu, ví dụ:

```
Trước khi trả kết quả, áp dụng skill "write-handoff-summary" để đóng gói theo
chuẩn handoff (status/conclusion/evidence/confidence).
```

Việc này giúp giảm token lặp lại ở cả 7 system prompt mà không mất thông tin —
đúng nguyên tắc "Lazy-load knowledge" và "Skill trước, Subagent sau" ở mục 2-3
của `04-token-efficiency.md`.

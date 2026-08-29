# TV360 FAST Multi-Agent System — Claude Code

Hệ thống 9 agent (1 orchestrator + 8 agent con) xây dựng báo cáo chiến lược
phát triển kênh FAST và hỗ trợ vận hành các kênh FAST của TV360.

## Cách bắt đầu

Mọi yêu cầu liên quan hệ thống FAST TV360 nên bắt đầu bằng cách gọi subagent
**`01-orchestrator`** — đây là entry point duy nhất, tự động route tới đúng
agent con theo pha (Strategy/Operations).

## Danh sách agent

| Subagent | Vai trò | Tạo file? |
|---|---|---|
| `01-orchestrator` | Điều phối + tổng hợp deliverable cuối | .docx, .pptx, .xlsx (bước 8b) |
| `02-market-researcher` | Nghiên cứu thị trường FAST quốc tế/khu vực | Không (đề xuất dữ liệu biểu đồ) |
| `03-data-analyst` | Phân tích dữ liệu TV360/VTVcab | .xlsx + biểu đồ |
| `04-strategy-planner` | Đề xuất chiến lược kênh/nội dung/danh mục | Không |
| `05-business-case-agent` | Chi phí, phương án kinh doanh, định nghĩa KPI chuẩn | .xlsx + biểu đồ |
| `06-ops-readiness-agent` | Quy trình, nguồn lực, công nghệ vận hành | Không |
| `07-scheduling-agent` | Khung phát sóng / lịch chi tiết hàng ngày | .xlsx |
| `08-kpi-reporting-agent` | Thu thập/đánh giá KPI theo bộ chuẩn | Không (đề xuất dữ liệu biểu đồ) |
| `09-content-director` | Phê duyệt từng phần (partial approval) | Không |

## Tài liệu tham chiếu bắt buộc

- `orchestration/plan-document.md` (PLAN.md) — kim chỉ nam TĨNH,
  `01-orchestrator` PHẢI đọc trước khi xử lý bất kỳ request nào.
- `orchestration/plan-tracking.md` — cấu trúc trạng thái ĐỘNG,
  `01-orchestrator` cập nhật xuyên suốt workflow.
- `knowledge/tv360-fast-glossary.md` — literacy nền dùng chung: thuật ngữ FAST,
  công thức tính chỉ số (mục 4), bối cảnh TV360 (mục 5), references nguồn đáng
  tin cậy (mục 7). MỌI agent phải tham chiếu để dùng chung 1 nền kiến thức.
- `skills/skill-write-handoff-summary.md`, `skills/skill-format-partial-approval.md`
  — 2 skill dùng chung khi agent con trả kết quả.

## Tạo file (.docx/.xlsx/.pptx)

Các agent có nhu cầu tạo file (`01-orchestrator`, `03-data-analyst`,
`05-business-case-agent`, `07-scheduling-agent`) PHẢI đọc đúng skill hệ thống
trước khi viết bất kỳ code nào:
- `.docx` → `/mnt/skills/public/docx/SKILL.md`
- `.xlsx` → `/mnt/skills/public/xlsx/SKILL.md`
- `.pptx` → `/mnt/skills/public/pptx/SKILL.md`

Không tự đoán API của docx-js/openpyxl/pptxgenjs — các skill này có gotchas cụ
thể (VD: hex màu không có `#` trong pptxgenjs, công thức Excel phải chạy
`recalc.py` mới có giá trị cache, docx-js mặc định khổ A4 không phải Letter).
Luôn verify output theo đúng quy trình QA của từng skill trước khi coi là
hoàn tất — không bỏ qua bước này dù đang gấp.

## Kiến trúc 2 pha

- **Pha Strategy**: xây báo cáo chiến lược đầy đủ, chạy tuần tự/song song theo
  `strategy_playbook` trong PLAN.md, kết thúc bằng `09-content-director` duyệt
  từng phần, rồi `01-orchestrator` xuất bộ file deliverable.
- **Pha Operations**: vận hành liên tục, chỉ gọi `07-scheduling-agent` và/hoặc
  `08-kpi-reporting-agent`, KHÔNG qua `09-content-director`.

## Ghi chú quan trọng

- Đây là bản render cho **Claude Code** (có code execution thật). Project này
  cũng có bản `rendered/claude-api/` cho Claude API/Project (KHÔNG tạo file
  thật, chỉ trả dữ liệu có cấu trúc) — 2 bản phục vụ 2 cách triển khai khác
  nhau, không dùng lẫn lộn.
- Spec nguồn trung lập platform nằm ở `agents/` (thư mục cha của project) —
  khi cần sửa logic nghiệp vụ, sửa ở đó trước rồi render lại cả 2 bản.

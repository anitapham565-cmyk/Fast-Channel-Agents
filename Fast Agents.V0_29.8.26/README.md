# TV360 FAST Multi-Agent System — Kiến trúc tổng thể

Hệ thống 9 agent phối hợp xây dựng báo cáo chiến lược phát triển kênh FAST
(Free Ad-supported Streaming TV) của TV360 và hỗ trợ vận hành các kênh FAST
trong quá trình triển khai.

**Nền tảng đích:** Ban đầu chỉ Claude API/Project. Kể từ đợt cập nhật 5 (yêu
cầu tạo file .docx/.xlsx/.pptx thật), project có **2 bản render song song**:
- `rendered/claude-api/` — Claude API/Project thuần, KHÔNG tạo file thật, chỉ
  trả dữ liệu có cấu trúc (text/JSON). Dùng khi tích hợp qua code driver tự viết.
- `rendered/claude-code/` — Claude Code (CÓ code execution thật), agent tự tạo
  và verify file .docx/.xlsx/.pptx theo skill hệ thống có sẵn. Đây là bản
  **được khuyến nghị nếu cần file tải về được** (yêu cầu ban đầu của Anita).

Xem `03-platform-mapping.md` mục 0 và mục 1b để hiểu khác biệt 2 nền tảng.

---

## 1. Sơ đồ kiến trúc

```
                    ┌─────────────────────────────┐
                    │  Agent #1 — Orchestrator     │
                    │  (TV360 FAST Orchestrator)   │
                    └──────────────┬────────────────┘
                                   │
                 ┌─────────────────┴─────────────────┐
                 │                                     │
          PHA STRATEGY                          PHA OPERATIONS
                 │                                     │
    ┌────────────┼────────────┐              ┌─────────┴─────────┐
    │            │             │              │                    │
 Agent #2    Agent #3          │           Agent #7            Agent #8
 (nghiên     (dữ liệu          │         (lịch phát sóng      (KPI định kỳ)
 cứu thị     TV360, song       │          hàng ngày)          KHÔNG qua #9
 trường,     song với #2)      │         KHÔNG qua #9
 song song                     │
 với #3)                       │
    └────────────┬────────────┘
                 │
            Agent #4
      (đề xuất chiến lược,
       tổng hợp #2 + #3)
                 │
    ┌────────────┴────────────┐
    │                          │
 Agent #5                  Agent #6
 (chi phí,                (vận hành,
 phương án                 nguồn lực,
 kinh doanh,               công nghệ)
 định nghĩa
 bộ KPI chuẩn)
    └────────────┬────────────┘
                 │
            Agent #7
      (khung phát sóng
       tổng thể — pha
       Strategy)
                 │
            Agent #8
      (KPI baseline dự
       kiến — pha Strategy)
                 │
                 ▼
       Agent #1 tổng hợp toàn bộ
       output #2-#8 thành 1 báo
       cáo duy nhất (giữ attribution)
                 │
                 ▼
            Agent #9
    (Giám đốc Nội dung — duyệt
     TỪNG PHẦN độc lập, per-agent-
     section: #2 đến #8)
                 │
     ┌───────────┴────────────┐
     │                          │
 TẤT CẢ approved          CÓ PHẦN rejected
     │                          │
 Xuất báo cáo cuối      Agent #1 route lại ĐỒNG THỜI
 cho Anita, kết thúc    mọi agent có phần bị từ chối
                        (retry_count chung +1)
                                │
                    retry_count = 3 mà vẫn có
                    phần rejected → DỪNG, báo
                    cáo thủ công cho Anita
```

## 2. Danh sách agent và vai trò

| # | Agent | Vai trò | Có gọi tool? | Pha hoạt động |
|---|---|---|---|---|
| 1 | TV360 FAST Orchestrator | Điều phối, tổng hợp, route retry | Không | Cả 2 pha |
| 2 | FAST Market Research Agent | Nghiên cứu thị trường FAST quốc tế/khu vực | Có (web_search, file_reader) | Strategy |
| 3 | TV360 Data Analysis Agent | Phân tích dữ liệu TV360/VTVcab | Có (file_reader, code_execution) | Strategy |
| 4 | FAST Strategy & Content Planning Agent | Đề xuất chiến lược kênh/nội dung/danh mục | Không | Strategy |
| 5 | FAST Business Case & KPI Definition Agent | Chi phí, phương án kinh doanh, ĐỊNH NGHĨA bộ KPI chuẩn | Không | Strategy |
| 6 | FAST Operations Readiness Agent | Quy trình, nguồn lực, công nghệ vận hành | Không | Strategy |
| 7 | FAST Scheduling Agent | Khung phát sóng (Strategy) / lịch chi tiết hàng ngày (Operations) | Không | Cả 2 pha |
| 8 | FAST KPI Reporting Agent | Thu thập/đánh giá KPI theo bộ chuẩn của #5 | Có (file_reader) | Cả 2 pha |
| 9 | Content Director Approval Agent | Phê duyệt từng phần (partial approval) | Không | Chỉ Strategy |

## 3. Các quyết định thiết kế quan trọng (đã chốt cùng Anita)

1. **2 pha vận hành tách biệt:** Strategy (xây báo cáo chiến lược ban đầu,
   chạy đầy đủ #2→#9) và Operations (vận hành liên tục, chỉ gọi #7 và/hoặc #8,
   KHÔNG qua Agent #9 duyệt). Agent #1 tự suy luận pha nào dựa trên từ khoá
   trong yêu cầu của Anita; nếu không rõ thì hỏi lại.

2. **Song song có kiểm soát:** #2+#3 chạy song song (độc lập với nhau);
   #5+#6 chạy song song (cùng dùng output #4 làm input). `max_parallel: 2`.

3. **Partial approval + retry theo vòng chung:** Agent #9 duyệt/từ chối
   TỪNG PHẦN độc lập (theo agent nguồn #2-#8), không phải duyệt/từ chối toàn bộ.
   Khi có phần bị từ chối, Agent #1 route lại ĐỒNG THỜI mọi agent có phần bị
   từ chối. `max_retries: 3` là **1 bộ đếm chung** cho cả báo cáo — không tách
   riêng theo từng phần. Hết 3 vòng mà vẫn còn phần chưa đạt → dừng, báo cáo
   thủ công cho Anita.

4. **Chuỗi phụ thuộc KPI (quan trọng, dễ nhầm):** Agent #5 là nơi DUY NHẤT
   định nghĩa bộ chỉ tiêu đo lường hiệu quả kênh (Views, DAU, MAU, HOV,
   Duration, Average Session Duration, DAU/MAU, HOV/MAU, doanh thu quảng
   cáo...). Agent #8 chỉ THU THẬP số liệu thực tế theo đúng bộ chuẩn này ở
   pha Operations — không được tự định nghĩa hoặc thêm chỉ số mới. Nếu Agent #8
   thấy cần chỉ số mới, phải báo lại Agent #1 để chuyển cho Agent #5 xem xét.

5. **FAST là đội ngũ vận hành riêng biệt**, không dùng chung nhân sự với
   VTVcab (đã xác nhận rõ trong hard_constraints của Agent #6). Nhưng hạ tầng
   công nghệ streaming thì TẬN DỤNG hạ tầng TV360 hiện có, chỉ đánh giá bổ
   sung/điều chỉnh, không xây mới toàn bộ.

6. **Danh sách nền tảng FAST ưu tiên theo dõi** (Agent #2): Plex, Samsung TV
   Plus, Pluto TV, The Roku Channel, Astro, Sooka, Tubi (FAST channel).

7. **Chưa có hệ thống kết nối dữ liệu/database thật ở giai đoạn này.** Agent #2,
   #3, #8 đều dùng cơ chế tạm thời: Anita/Agent #1 upload file trực tiếp vào
   hội thoại. Khi TV360 có hệ thống lưu trữ/API nội bộ chính thức, cần cập
   nhật lại Tầng C (Tool Specification) của 3 agent này — hiện đang dùng
   `file_reader` tạm thời thay vì kết nối MCP/API thật.

8. **Ranh giới nội dung/khung phát sóng:** Agent #4 chỉ đưa định hướng khung
   TỔNG THỂ (VD: "giờ vàng ưu tiên thể loại X"), không lập khung chi tiết theo
   giờ. Agent #7 mới là nơi lập khung chi tiết (Strategy) và lịch hàng ngày
   (Operations). Agent #4 cũng KHÔNG đề xuất danh sách show/chương trình cụ
   thể — chỉ ở mức định hướng loại nội dung.

## 4. Bổ sung mới (đợt cập nhật 2): Skills, Knowledge, Plan Tracking, Token Efficiency

### 4.1 Skills dùng chung (phát hiện qua rà soát 9 agent)

- **`write-handoff-summary`** — dùng bởi Agent #2 đến #8. Đóng gói kết quả
  thành handoff chuẩn (status/conclusion/evidence/confidence) trước khi trả
  về Agent #1. Trước khi tách skill này, cả 7 agent tự viết lại nguyên văn
  cấu trúc handoff trong system prompt — đây là phần lặp lại rõ nhất trong
  toàn bộ hệ thống.
- **`format-partial-approval`** — chỉ dùng bởi Agent #9. Đóng gói kết quả
  duyệt theo per-agent-section (mỗi phần #2-#8 có status độc lập). Dù chỉ 1
  agent dùng (thường không đủ điều kiện tách Skill theo mục 2 của
  `05-skill-design-guide.md`), vẫn tách vì đây là điểm dễ lỗi nhất hệ thống
  và cần version/test riêng.

Xem chi tiết đầy đủ (input/output contract, quality_checks) tại
`skills/skill-write-handoff-summary.md` và `skills/skill-format-partial-approval.md`.

### 4.2 Kiến thức nền tảng dùng chung

File `knowledge/tv360-fast-glossary.md` chứa thuật ngữ FAST (dayparting,
prime-time, CPM, EPG, ad-insertion...), định nghĩa chỉ số (DAU/MAU/HOV...),
và các giả định nền đã xác nhận với Anita (TV360 đã có kênh FAST + hạ tầng
sẵn có, đội ngũ FAST riêng biệt VTVcab, giới hạn không xử lý pháp luật phát
sóng). Agent #2 đến #8 đều tham chiếu file này qua `knowledge_references` ở
Tầng A — tránh mỗi agent tự định nghĩa lại thuật ngữ khác nhau.

**Lưu ý:** file này KHÔNG chứa số liệu thị trường hiện tại (đó vẫn là việc
web_search của Agent #2) — chỉ chứa kiến thức nền ổn định theo thời gian.

### 4.3 Plan Tracking (cơ chế theo dõi tiến độ, KHÁC với Plan chiến lược)

File `orchestration/plan-tracking.md` định nghĩa 1 cấu trúc trạng thái độc
lập mà Agent #1 duy trì và cập nhật xuyên suốt workflow — theo dõi agent nào
đã chạy/đang chạy/thất bại, vòng duyệt thứ mấy, lý do dừng nếu có. Đây là
tầng bổ sung ngoài schema gốc 00-generic-schema.md (chưa có sẵn tầng này),
gắn trực tiếp vào Tầng B (SOP) và Tầng F (Orchestration) của Agent #1.

**Phân biệt quan trọng — 2 khái niệm "Plan" khác nhau, dễ nhầm:**
- **Plan chiến lược** (lộ trình triển khai kênh FAST ngoài đời thực — giai
  đoạn, milestone kinh doanh) = output "Kế hoạch triển khai" của **Agent #4**.
  Không phải file riêng.
- **Plan Tracking** (tài liệu mục này) = trạng thái vận hành của chính hệ
  thống 9 agent (kỹ thuật, không phải nội dung nghiệp vụ).

Theo yêu cầu của Anita, plan_tracking được lưu ở **cả 2 nơi**: (1) truyền
trực tiếp trong context của Agent #1 qua các lượt gọi API, và (2) ghi snapshot
vào storage ngoài (do code driver thực hiện, không phải chính model Claude)
để tra cứu lại sau hoặc phục hồi khi phiên bị ngắt.

### 4.4 Chính sách hạn chế token (bổ sung Tầng F của Agent #1)

Đã thêm `token_efficiency_policy` vào Tầng F của Agent #1
(`agents/agent-01-orchestrator.md`), gồm: bắt buộc agent con dùng Skill
`write-handoff-summary` (không nhận văn bản dài chưa tóm tắt), giới hạn
conclusion ≤3 câu, chỉ truyền context tối thiểu cần thiết khi gọi/retry agent
con, và plan_tracking chỉ lưu trạng thái ngắn gọn (không nhúng nội dung
nghiệp vụ đầy đủ — nội dung đầy đủ nằm trong handoff_package).

### 4.5 Plan tổng thể — SỬA LẠI bản chất (đợt cập nhật 4, quan trọng)

**Đính chính:** ở đợt cập nhật trước, PLAN.md được mô tả sai là "output động
trích xuất từ Agent #4". Anita đã làm rõ lại: PLAN.md là **kim chỉ nam TĨNH**
(playbook cố định, viết 1 lần, KHÔNG đổi theo từng request) mà **Agent #1 đọc
TRƯỚC khi bắt đầu điều phối** để biết đúng thứ tự/quy tắc gọi agent — không
phải tài liệu do Agent #4 sinh ra.

Xem nội dung đầy đủ tại `orchestration/plan-document.md` (đã viết lại hoàn
toàn) — gồm `strategy_playbook` (thứ tự 6 bước cho pha Strategy), 
`operations_playbook` (2 bước độc lập cho pha Operations), và
`global_constraints` (max_parallel, max_retries, hub_and_spoke...).

**Phân biệt 3 khái niệm "Plan" — bản cập nhật:**
| Khái niệm | Là gì | Tĩnh/Động | Ở đâu |
|---|---|---|---|
| Plan chiến lược (nội dung) | Milestone/giai đoạn triển khai kênh FAST thực tế | Động (khác nhau mỗi request) | Vẫn là output "Kế hoạch triển khai" của Agent #4, KHÔNG tách ra file riêng nữa |
| PLAN.md (plan-document) | Playbook/kim chỉ nam quy trình điều phối 9 agent | **TĨNH** (cố định, Agent #1 chỉ đọc) | `orchestration/plan-document.md` |
| Plan Tracking | Trạng thái vận hành kỹ thuật thực tế của 1 request cụ thể | Động (1 bản/request) | `orchestration/plan-tracking.md`, phải khớp cấu trúc với PLAN.md |

Agent #1 giờ có **step 0** mới trong Tầng B: đọc PLAN.md trước khi xử lý bất
kỳ request nào. Agent #4 **không còn liên quan** tới việc tạo/ghi PLAN.md.

### 4.6 Model và Tool chuẩn hóa cho toàn bộ 9 agent

- **Model:** tất cả 9 agent dùng `claude-sonnet-5`.
- **Tools:** cả 9 agent được cấp bộ 4 tool chuẩn: `WebSearch`, `WebFetch`,
  `Read`, `Write`. Agent không có nhu cầu nghiệp vụ dùng 1 tool nào đó vẫn
  khai báo tool đó nhưng đánh dấu rõ `when_NOT_to_use`.
- Agent #3 giữ thêm `code_execution` ngoài 4 tool chuẩn.

### 4.7 Literacy nền sâu hơn + References nguồn đáng tin cậy (đợt cập nhật 4)

**Công thức tính cụ thể (yêu cầu mới):** `tv360-fast-glossary.md` mục 4 đã
được viết lại thành bảng có **công thức tính rõ ràng** cho từng chỉ số (DAU/
MAU, HOV/MAU, CPM, Ad load, Fill rate, Average Session Duration...) — không
chỉ định nghĩa suông như bản trước. Agent #3 (phân tích dữ liệu) và Agent #8
(báo cáo KPI) bắt buộc dùng đúng công thức này, không tự suy diễn cách tính
khác. Đã bổ sung `self_check_before_output` cho Agent #3 để kiểm tra điểm
này.

**References — nguồn tham chiếu đáng tin cậy (mục mới, `tv360-fast-glossary.md`
mục 7):** danh sách nguồn bên ngoài Agent #2 ưu tiên khi web_search, chia theo
độ tin cậy:
- 7.1 Trang chính thức nền tảng FAST (Pluto TV, Tubi, Samsung TV Plus, Roku,
  Plex, Astro, Sooka) — ưu tiên cao nhất.
- 7.2 Báo cáo thị trường uy tín (Nielsen, eMarketer, Parks Associates, Ampere
  Analysis, MoffettNathanson, Antenna).
- 7.3 Báo chí chuyên ngành (Variety, Hollywood Reporter, Broadcasting & Cable,
  Digiday, AdExchanger).
- 7.4 Nguồn KHÔNG ưu tiên (diễn đàn, blog cá nhân không rõ nguồn gốc, bài
  không ghi ngày tháng) — tuyệt đối không dùng làm căn cứ.

Đã thêm `citation_rule` áp dụng cho mọi agent (không riêng Agent #2): thứ tự
ưu tiên nguồn, cách xử lý khi 2 nguồn mâu thuẫn. Agent #2 đã cập nhật
`hard_constraints` để bám theo citation_rule; Agent #9 đã cập nhật
`knowledge_references` để dùng danh sách này khi đánh giá độ tin cậy phần
nghiên cứu thị trường lúc duyệt.

### 4.8 Output file .docx/.xlsx/.pptx (đợt cập nhật 5 — thay đổi nền tảng)

**Vấn đề phát hiện khi triển khai yêu cầu này:** tạo file thật (.docx/.xlsx/
.pptx) cần code execution (chạy Python/Node, thao tác filesystem) — năng lực
này KHÔNG tồn tại trong Claude API JSON thuần (thiết kế gốc của 9 agent). Sau
khi giải thích sự khác biệt, Anita xác nhận chuyển sang **Claude Code/Agent
SDK** để agent thật sự tạo và verify file theo đúng quy trình 3 skill hệ thống
đã có sẵn (`/mnt/skills/public/docx`, `/mnt/skills/public/xlsx`,
`/mnt/skills/public/pptx`).

**Phân bổ file theo agent (đã xác nhận với Anita):**
| Agent | File tạo | Ghi chú |
|---|---|---|
| `tv360-fast-orchestrator` | .docx (báo cáo tổng hợp + phụ lục), .pptx (slide), .xlsx (bảng tổng hợp) | Chỉ tạo ở bước 8b, sau khi TẤT CẢ phần được `content-director` duyệt |
| `market-researcher` | Không tự tạo | Đề xuất dữ liệu + gợi ý biểu đồ để orchestrator dùng khi tạo .docx |
| `data-analyst` | .xlsx + biểu đồ | Phân tích dữ liệu TV360/VTVcab |
| `strategy-planner` | Không | Chỉ trả nội dung text |
| `business-case-agent` | .xlsx + biểu đồ | Mô hình tài chính, theo quy ước màu sắc chuẩn của skill xlsx |
| `ops-readiness-agent` | Không | Chỉ trả nội dung text |
| `scheduling-agent` | .xlsx | Lịch phát sóng, cả 2 pha |
| `kpi-reporting-agent` | Không tự tạo | Đề xuất dữ liệu + gợi ý biểu đồ KPI để orchestrator dùng |
| `content-director` | Không | Chỉ đánh giá/duyệt |

**Kiến trúc:** đa số agent con chỉ trả dữ liệu có cấu trúc, KHÔNG tự tạo file
tổng hợp cuối — `tv360-fast-orchestrator` là nơi duy nhất tổng hợp TẤT CẢ định
dạng file cuối cùng (bước 8b), đồng thời đính kèm nguyên trạng các file .xlsx
riêng đã nhận từ `data-analyst`/`business-case-agent`/`scheduling-agent`.

**Bản Claude Code là bản chính thức cho yêu cầu này** — xem
`rendered/claude-code/` (9 file `.md` trong `.claude/agents/` + `CLAUDE.md`
hướng dẫn tổng quan). Bản `rendered/claude-api/` được giữ lại cho trường hợp
Anita muốn tích hợp qua code driver tự viết (không cần Claude tự tạo file),
nhưng **chưa được cập nhật thêm gì cho yêu cầu file** — nếu dùng bản này, cần
tự dựng 1 "File Generation Service" riêng bên ngoài nhận dữ liệu JSON từ agent
và dựng file, KHÔNG có trong phạm vi đã triển khai.

### 4.9 Đặt tên agent khoa học hơn — có số thứ tự (đợt cập nhật 6)

Theo yêu cầu của Anita, 9 agent trong **bản `rendered/claude-code/`** đã được
đổi tên (cả tên file lẫn field `name` trong frontmatter) để có số thứ tự #1-#9
ở đầu, dễ theo dõi và khớp đúng thứ tự vai trò trong hệ thống:

| Số | Tên cũ | Tên mới |
|---|---|---|
| 1 | `tv360-fast-orchestrator` | `01-orchestrator` |
| 2 | `market-researcher` | `02-market-researcher` |
| 3 | `data-analyst` | `03-data-analyst` |
| 4 | `strategy-planner` | `04-strategy-planner` |
| 5 | `business-case-agent` | `05-business-case-agent` |
| 6 | `ops-readiness-agent` | `06-ops-readiness-agent` |
| 7 | `scheduling-agent` | `07-scheduling-agent` |
| 8 | `kpi-reporting-agent` | `08-kpi-reporting-agent` |
| 9 | `content-director` | `09-content-director` |

Đã cập nhật đồng bộ: tên file, field `name:` trong frontmatter, mọi tham chiếu
chéo trong `description` và nội dung body (backtick lẫn văn xuôi thường) của cả
9 file, và bảng danh sách agent trong `CLAUDE.md`.

**Lưu ý phạm vi:** việc đổi tên này chỉ áp dụng cho `rendered/claude-code/` —
đây là bản Anita xác nhận đang dùng. Các file spec nguồn (`agents/*.md`, dùng
"Agent #1", "Agent #2"...) và bản `rendered/claude-api/` (dùng tên đầy đủ như
"TV360 FAST Orchestrator") vẫn giữ quy ước đặt tên cũ, vì không phải bản đang
triển khai. Nếu cần đồng bộ tên ở các bản đó, cần yêu cầu riêng.

## 5. Giới hạn đã biết / việc cần làm tiếp

- **Chưa có tool kết nối database/CRM/BI thật** cho Agent #2, #3, #8 — đang
  dùng file upload thủ công làm giải pháp tạm. Cần bổ sung khi TV360 có hạ
  tầng dữ liệu chính thức.
- **Danh sách chỉ tiêu KPI** (Views, DAU, MAU, HOV...) được Anita xác nhận là
  "sẽ tiếp tục update" — cần rà lại `hard_constraints` của Agent #5 và Agent #8
  định kỳ khi có chỉ tiêu mới.
- **Quy định pháp luật phát sóng VN** (kiểm duyệt, khung giờ theo luật...)
  KHÔNG nằm trong phạm vi Agent #7 theo quyết định của Anita — cần xác nhận
  bộ phận nào chịu trách nhiệm xử lý việc này trước khi go-live thực tế.
- **Tiêu chí duyệt của Agent #9** hiện ở mức định tính (tính nhất quán logic
  nội bộ, mức độ có căn cứ) — chưa có tiêu chí nghiệp vụ định lượng cụ thể
  (VD: ngưỡng ROI tối thiểu). Có thể bổ sung sau nếu Anita muốn kiểm soát
  chặt hơn.

## 6. Cấu trúc thư mục

```
tv360-fast-project/
├─ README.md                          (file này)
├─ agents/                             (spec nguồn — schema 5+1 tầng, trung lập platform)
│  ├─ agent-01-orchestrator.md         (đã cập nhật: plan_tracking_rule + token_efficiency_policy)
│  ├─ agent-02-market-research.md      (đã cập nhật: tham chiếu skill + knowledge)
│  ├─ agent-03-data-analysis.md
│  ├─ agent-04-strategy-content.md
│  ├─ agent-05-business-case-kpi.md
│  ├─ agent-06-operations-readiness.md
│  ├─ agent-07-scheduling.md
│  ├─ agent-08-kpi-reporting.md
│  └─ agent-09-approval.md             (đã cập nhật: dùng skill format-partial-approval)
├─ skills/                             (MỚI — Skill dùng chung, theo 06-skill-schema.md)
│  ├─ skill-write-handoff-summary.md
│  └─ skill-format-partial-approval.md
├─ knowledge/                          (MỚI — Knowledge reference, không phải Skill/Agent)
│  └─ tv360-fast-glossary.md
├─ orchestration/                      (cơ chế điều phối bổ sung ngoài schema gốc)
│  ├─ plan-tracking.md                 (trạng thái vận hành kỹ thuật 9 agent, ĐỘNG)
│  └─ plan-document.md                 (PLAN.md — kim chỉ nam quy trình, TĨNH)
└─ rendered/
   ├─ claude-api/                      (Claude API/Project — KHÔNG tạo file thật, chỉ trả dữ liệu có cấu trúc)
   │  ├─ agent-01-orchestrator.md
   │  ├─ agent-02-market-research.md
   │  ├─ agent-03-data-analysis.md
   │  ├─ agent-04-strategy-content.md
   │  ├─ agent-05-business-case-kpi.md
   │  ├─ agent-06-operations-readiness.md
   │  ├─ agent-07-scheduling.md
   │  ├─ agent-08-kpi-reporting.md
   │  └─ agent-09-approval.md
   └─ claude-code/                     (bản chính thức cho yêu cầu tạo file .docx/.xlsx/.pptx)
      ├─ CLAUDE.md                     (hướng dẫn tổng quan project)
      ├─ .claude/agents/               (9 subagent, ĐÃ ĐỔI TÊN có số thứ tự #1-#9)
      │  ├─ 01-orchestrator.md
      │  ├─ 02-market-researcher.md
      │  ├─ 03-data-analyst.md
      │  ├─ 04-strategy-planner.md
      │  ├─ 05-business-case-agent.md
      │  ├─ 06-ops-readiness-agent.md
      │  ├─ 07-scheduling-agent.md
      │  ├─ 08-kpi-reporting-agent.md
      │  └─ 09-content-director.md
      ├─ orchestration/                (copy plan-document.md + plan-tracking.md)
      ├─ knowledge/                    (copy tv360-fast-glossary.md)
      └─ skills/                       (copy 2 skill dùng chung)
```

## 7. Trạng thái render — ĐÃ ĐỒNG BỘ (cập nhật mới nhất)

Toàn bộ 9 file trong `rendered/claude-api/` đã được render lại đầy đủ, đồng bộ
với `agents/` mới nhất (bao gồm cả 3 đợt cập nhật: skill/knowledge/plan-tracking,
và model/tools/plan-document). Đã kiểm tra:
- Cả 9 file có `"model": "claude-sonnet-5"`.
- Cả 9 file có đủ 4 tool `WebSearch`, `WebFetch`, `Read`, `Write` ở dạng JSON
  `input_schema` đầy đủ (không còn tên tool cũ `web_search`/`file_reader` viết
  thường, không còn `"tools": "N/A"`).
- Toàn bộ JSON trong 9 file đã validate cú pháp hợp lệ.
- Agent #1 đã có bước 3b (trích xuất PLAN.md) và `token_efficiency_policy`/
  `plan_tracking_rule` đầy đủ trong system prompt.

**Lưu ý khi triển khai thật:** với các agent không có nhu cầu nghiệp vụ dùng 1
số tool trong bộ 4 chuẩn (VD Agent #4 không cần `WebSearch`), file render vẫn
khai báo đủ tool đó (theo đúng yêu cầu đồng bộ của Anita) nhưng đánh dấu rõ
`description` là "KHÔNG dùng trong vận hành bình thường". Nếu muốn tối ưu token
triệt để hơn theo `04-token-efficiency.md` (tool list càng hẹp càng tốt), có thể
loại bỏ tool không dùng khỏi từng agent cụ thể khi triển khai thật — đánh đổi là
mất tính đồng nhất "cả 9 agent cùng 1 bộ tool" mà Anita yêu cầu ban đầu.

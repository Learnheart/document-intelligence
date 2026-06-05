---
author: lytinh358@gmail.com
date: 2026-06-05
status: draft
agents: idp
summary: Từ báo cáo thị trường Document Intelligence → xác định scope và các phần phải xử lý để triển khai hệ IDP (phạm vi outsource project)
---

# Scope & Kế hoạch triển khai hệ Document Intelligence (IDP)

> **Mục đích:** Chuyển kỳ vọng thị trường (xem [Phụ lục — nguồn](#phụ-lục-a--tóm-tắt-dữ-liệu-thị-trường)) thành **phạm vi giao hàng cụ thể** và **danh mục công việc phải xử lý** để xây hệ IDP.
> **Altitude:** Tài liệu này viết cho một **dự án outsource** — giao một hệ IDP chạy được cho 1–3 use case ROI cao, KHÔNG phải dựng một chương trình governance/AI cấp doanh nghiệp. Ranh giới trách nhiệm với client được nêu tường minh ở [§8](#8-phụ-thuộc--bàn-giao-handoff).
> **Liên kết:** Kiến trúc xử lý → `workflow.md`; vòng đời & quản trị dữ liệu → `governance.md`. Tài liệu này là lớp *scope/delivery* ngồi trên hai file đó.

---

## Mục lục

1. Tóm tắt điều hành
2. Kỳ vọng thị trường → Yêu cầu hệ thống (ánh xạ)
3. Phạm vi (in-scope / out-of-scope / giả định)
4. Các phần phải xử lý để triển khai (work breakdown)
5. KPI nghiệm thu
6. Quyết định nền tảng: build-vs-buy & cảnh báo "agentic"
7. Lộ trình triển khai theo giai đoạn
8. Phụ thuộc & bàn giao (handoff)
9. Rủi ro & giảm thiểu
- Phụ lục A — Tóm tắt dữ liệu thị trường

---

## 1. Tóm tắt điều hành

Thị trường IDP đang chuyển từ thử nghiệm sang triển khai diện rộng, CAGR hai con số (phần lớn 26–33%), với BFSI dẫn đầu và y tế tăng nhanh nhất. Nhưng kỳ vọng đã **đổi chất**: không còn là "số hóa giấy tờ" mà là hệ **hiểu ngữ cảnh, tự động hóa đầu-cuối, tích hợp thẳng vào ERP/CRM, có kiểm soát tuân thủ và ROI đo được**.

Ba sự thật chi phối cách ta định phạm vi:

1. **Tích hợp là nơi ROI sống hoặc chết.** Trích xuất mà không có API hai chiều với hệ hạ nguồn → nhân viên copy JSON thủ công → triệt tiêu tiết kiệm. Tích hợp **phải in-scope từ Giai đoạn 1**, không để sau.
2. **"Agentic" đang bị thổi phồng.** Gartner dự báo >40% dự án agentic bị hủy tới 2027 vì chi phí leo thang và ROI mơ hồ; nhiều use case gắn nhãn "agentic" thực ra không cần. → Ta khởi đầu bằng **luồng tất định + full HITL** (đúng `workflow.md` ADR-10/11), chỉ thêm agentic ở nơi ROI đã được chứng minh.
3. **Bắt đầu hẹp mới thắng.** Doanh nghiệp do dự vì ROI mơ hồ. → Chọn **1–3 use case "dễ thắng"** (hóa đơn/AP, KYC onboarding, claims), chốt **KPI đo được TRƯỚC khi build**, rồi mới mở rộng.

**Luận điểm phạm vi:** Giao một hệ IDP **lõi tất định + full HITL + tích hợp ERP/CRM + governance mức dự án + đo KPI**, cho **1 use case ở Giai đoạn 1**, mở rộng use case và năng lực (VLM/template-free, confidence-gating) ở các giai đoạn sau. Phần kiến trúc đã có sẵn trong `workflow.md`/`governance.md`; tài liệu này chốt *làm bao nhiêu, theo thứ tự nào, ai chịu phần nào*.

---

## 2. Kỳ vọng thị trường → Yêu cầu hệ thống (ánh xạ)

Mỗi nhóm kỳ vọng từ báo cáo được dịch thành yêu cầu cụ thể, kèm trạng thái: đã có trong thiết kế hiện tại (`workflow.md`/`governance.md`) hay là **gap phải xây**.

| # | Kỳ vọng thị trường | Yêu cầu hệ thống cụ thể | Trạng thái thiết kế |
|---|---|---|---|
| 1 | Độ chính xác cao + STP đo được | Trích xuất field + confidence + chấm KPI (STP, first-pass yield, lỗi/1.000, cycle time) | Trích xuất/validation có (`workflow §4.9`); **đo KPI là gap** (cần lớp metrics — `governance §9` mới ở mức chỉ số, chưa có dashboard) |
| 2 | Hiểu phi cấu trúc, không phụ thuộc template (zero-shot) | OCR + VLM/LLM đọc layout lạ, đa ngôn ngữ, viết tay | Có ở `workflow §4.6` (VLM) nhưng là **Giai đoạn 2**; Giai đoạn 1 dùng OCR+LLM cho tài liệu cấu trúc |
| 3 | Tích hợp đầu-cuối ERP/CRM (API hai chiều) | Integration layer: nhận document, trả JSON/XML có cấu trúc vào endpoint hạ nguồn, idempotent, reconciliation | **Gap lớn** — `workflow.md` chỉ mô tả pipeline nội bộ, chưa có lớp API/contract ngoài |
| 4 | Bảo mật, quản trị, kiểm toán, drift | PII firewall, audit trail, lineage, drift detection | Có nhiều: `workflow §8` (bảo mật) + `governance §4–§6` (lineage, drift) — mức dự án |
| 5 | Human-in-the-loop + hàng đợi ngoại lệ | UI duyệt cạnh tài liệu gốc; routing ngoại lệ; pre-fill/highlight/crop | UI có ở `workflow §4.10`; **vận hành hàng đợi/routing reviewer là gap** |
| 6 | Mở rộng + chi phí linh hoạt (cloud/hybrid, pay-as-you-go) | Autoscale theo queue depth; deploy cloud/hybrid theo tier | Có ở `workflow §6, §9` |
| 7 | ROI nhanh, đo được (3–6 tháng) | Baseline KPI + so sánh trước/sau; use case hẹp | Quy trình — **chốt ở tài liệu này** ([§5](#5-kpi-nghiệm-thu), [§7](#7-lộ-trình-triển-khai-theo-giai-đoạn)) |
| 8 | Tùy biến không cần kỹ năng ML | Định nghĩa schema/loại tài liệu bằng config + vài mẫu gán nhãn | **Gap** — chưa có schema/document-type management plane |

**Đọc bảng này:** phần lõi *xử lý* và *bảo mật* đã được thiết kế tốt. Bốn gap phải xây để chạm kỳ vọng thị trường là **(3) Integration layer, (5) Exception/HITL operations, (1) KPI measurement, (8) Schema management** — đây chính là trọng tâm của [§4](#4-các-phần-phải-xử-lý-để-triển-khai-work-breakdown).

---

## 3. Phạm vi

### 3.1. In-scope (dự án giao)

**Use case (chọn theo ROI, làm tuần tự — không song song):**

| Ưu tiên | Use case | Vì sao | Loại tài liệu |
|---|---|---|---|
| Giai đoạn 1 | **Hóa đơn / Accounts Payable** | ROI rõ nhất, tài liệu bán cấu trúc, có STP cho hóa đơn "sạch" | Hóa đơn, PO, biên nhận |
| Giai đoạn 2 | **KYC onboarding** | Nhu cầu BFSI cao, gắn tuân thủ | CMND/CCCD/hộ chiếu, đăng ký KD, sao kê |
| Tùy chọn / sau | **Claims bảo hiểm** | Giá trị cao nhưng đa dạng biểu mẫu → phức tạp hơn | Biểu mẫu claim, hồ sơ y tế/sửa chữa |

> Khởi đầu **một** use case (hóa đơn/AP). Use case 2 chỉ mở khi use case 1 đạt KPI nghiệm thu.

**Năng lực chức năng in-scope:**

- Ingestion đa nguồn + tiền xử lý + malware scan (`workflow §4.1`).
- PII firewall trước model (`workflow §4.2`).
- Phân loại loại tài liệu + (Giai đoạn 2) phân đoạn vùng (`workflow §4.3`).
- Trích xuất: **Giai đoạn 1** OCR + LLM cho field; **Giai đoạn 2** thêm VLM + sub-extractor bảng (`workflow §4.5–4.8`).
- Validation theo schema + business rule + confidence (`workflow §4.9`).
- **Full HITL** duyệt + UI cạnh-tài-liệu + **hàng đợi ngoại lệ** (`workflow §4.10` + lớp vận hành mới ở [§4-E](#e-hitl--hàng-đợi-ngoại-lệ)).
- **Integration layer**: API submit/status/result + đẩy JSON có cấu trúc vào ERP/CRM ([§4-F](#f-integration-layer-erpcrm)).
- Governance mức dự án: PII sanitize log, event schema, lineage cơ bản, retention, drift (`governance §4–§6`).
- **Đo KPI**: STP, first-pass yield, lỗi/1.000, cycle time ([§5](#5-kpi-nghiệm-thu)).
- Deploy cloud/hybrid theo tier (`workflow §9`).

**Ngôn ngữ:** Tiếng Việt (gồm dấu phụ + viết tay cơ bản) + tiếng Anh. Ngôn ngữ khác: out-of-scope Giai đoạn đầu.

### 3.2. Out-of-scope (client/doanh nghiệp tự sở hữu)

Liệt kê tường minh để tránh mơ hồ trách nhiệm — nhất quán với altitude outsource (xem `governance.md`, và memory dự án về scope governance):

- **Agentic orchestration đầy đủ** (vòng tự sửa đa bước, multi-agent) — để dành, chỉ làm khi ROI chứng minh (`workflow §13` Giai đoạn 4).
- **Machine unlearning / xóa lan tới model & embedding** (GDPR Art.17 ở tầng model) — chỉ bật nếu client ràng buộc hợp đồng.
- **Chương trình governance tổ chức**: RACI, data steward, change-approval board, responsible-AI program toàn doanh nghiệp.
- **Multi-tenancy quy mô doanh nghiệp**, data residency/sovereignty đa vùng, FinOps chargeback.
- **Sửa đổi hệ hạ nguồn** (ERP/CRM): client cung cấp endpoint nhận payload; ta tích hợp tới đó, không xây/sửa hệ hạ nguồn.
- **Vận hành production dài hạn** (on-call 24/7, SLA vận hành), trừ khi có hợp đồng bảo trì riêng.
- **Đội ngũ reviewer HITL**: ta giao *công cụ + quy trình* duyệt; nhân sự duyệt do client bố trí (trừ khi thỏa thuận khác).

### 3.3. Giả định (assumptions)

Hệ chạy đúng phụ thuộc các điều kiện client phải đáp ứng:

- Client cung cấp **bộ tài liệu mẫu** đại diện (đủ phủ định dạng/vendor) trước khi build.
- Client **chốt KPI nghiệm thu** ([§5](#5-kpi-nghiệm-thu)) và ngưỡng chấp nhận TRƯỚC khi triển khai.
- Client cung cấp **endpoint API hạ nguồn** (ERP/CRM) + tài khoản cloud/hạ tầng + credentials theo least-privilege.
- Tài liệu đầu vào đạt chất lượng tối thiểu (ví dụ scan ≥ 200 DPI); dưới ngưỡng → đẩy hàng đợi ngoại lệ, không tính vào KPI tự động.
- Khung tuân thủ áp dụng (GDPR/PDPA/SBV…) được client xác nhận từ đầu để cấu hình tier đúng.

---

## 4. Các phần phải xử lý để triển khai (work breakdown)

Mỗi hạng mục ghi rõ: **tái dùng** từ thiết kế có sẵn hay **xây mới**, và độ phức tạp tương đối (T/B/C = Thấp/Trung bình/Cao).

### A. Ingestion & tiền xử lý
- Tiếp nhận đa nguồn, chuẩn hóa, tách trang, khử nghiêng/nhiễu, malware scan, quarantine. **Tái dùng** `workflow §4.1`. Độ phức tạp: **B**.
- Kiểm tra chất lượng đầu vào (DPI, định dạng) → gate sớm. **Xây mới (nhỏ)**. **T**.

### B. Phân loại & phân đoạn
- Phân loại loại tài liệu (hóa đơn vs PO vs biên nhận). **Xây mới** (Giai đoạn 1, classifier nhẹ). **B**.
- Phân đoạn vùng (text/bảng/chữ ký). **Tái dùng** `workflow §4.3` — **Giai đoạn 2**. **C**.

### C. Trích xuất (OCR + LLM/VLM + sub-extractors)
- Đường OCR + LLM trích field (Giai đoạn 1, đáp ứng kỳ vọng template-free ở mức LLM). **Tái dùng** `workflow §4.5, §4.9`. **B**.
- Đường VLM cho layout phức tạp/viết tay (Giai đoạn 2). **Tái dùng** `workflow §4.6`. **C**.
- Sub-extractor bảng (claims/hóa đơn nhiều line-item). **Tái dùng** `workflow §4.7`. **C**.

### D. Validation & Schema/Business-rule management
- Áp business rule + chuẩn hóa (ngày/tiền) + schema validation + confidence. **Tái dùng** `workflow §4.9`. **B**.
- **Schema/document-type management plane** (định nghĩa field/rule bằng config + vài mẫu, không cần kỹ năng ML — kỳ vọng #8). **Xây mới — gap**. **C**.

### E. HITL & hàng đợi ngoại lệ
- UI duyệt cạnh tài liệu gốc, pre-fill/highlight confidence thấp/crop nguồn. **Tái dùng** `workflow §4.10`. **B**.
- **Vận hành hàng đợi ngoại lệ**: routing ca tới reviewer, ưu tiên, trạng thái, escalation. **Xây mới — gap** (đúng kỳ vọng #5 ở quy mô). **C**.
- Bắt **diff predicted-vs-corrected** làm golden data (nối `governance §3.1, §7.1`). **Tái dùng/cấu hình**. **T**.

### F. Integration layer (ERP/CRM)
- **API ngoài**: submit (202 Accepted) + status + result; webhook/callback; auth (OAuth/OIDC/API key/mTLS); idempotency-key client. **Xây mới — gap lớn nhất**. **C**.
- Đẩy payload JSON/XML có cấu trúc vào endpoint hạ nguồn; xử lý downstream từ chối + reconciliation (`downstream_status`). **Xây mới**. **C**.
- Kênh thông báo hoàn thành (WebSocket/SSE/polling). **Tái dùng** `workflow §6.5`. **B**.

### G. Governance & bảo mật (mức dự án)
- PII firewall + sanitize log + sensitivity label. **Tái dùng** `workflow §8.4` + `governance §5`. **B**.
- Event schema + version, lineage cơ bản (data→feature→model), retention/tiering. **Tái dùng** `governance §4`. **B**.
- Audit trail + RBAC + MFA cho HITL. **Tái dùng** `workflow §8.3, §8.10`. **B**.

### H. Observability & đo KPI
- Metrics vận hành: latency, cost/page, %VLM, %HITL, queue depth. **Tái dùng** `workflow §10`. **B**.
- **Dashboard KPI nghiệp vụ**: STP, first-pass yield, lỗi/1.000, cycle time — baseline + theo dõi. **Xây mới — gap** (kỳ vọng #1, #7). **B**.
- Drift/anomaly + runbook. **Tái dùng** `governance §6`. **B**.

### I. Deploy & mô hình chi phí
- Cloud/hybrid theo tier, autoscale theo queue depth, tách mạng. **Tái dùng** `workflow §9`. **B**.
- Mô hình pay-as-you-go + giám sát chi phí VLM (cache theo hash). **Tái dùng** `workflow §6.6`. **B**.

> **Tổng kết gap phải xây mới:** Schema management (D), Exception-queue operations (E), Integration layer (F), KPI dashboard (H). Đây là phần *delivery risk* cao nhất — ưu tiên estimate kỹ.

---

## 5. KPI nghiệm thu

Bài học thị trường: **chốt KPI đo được với mọi bên TRƯỚC khi build**. Bảng dưới là khung; con số mục tiêu phải client ký nhận ở Giai đoạn 0.

| Nhóm | KPI | Mục tiêu tham khảo | Ghi chú |
|---|---|---|---|
| Tự động hóa | **Straight-through processing (STP)** | 60–80% hóa đơn "sạch" (Giai đoạn 1) | Tăng dần khi tích lũy golden data |
| Chất lượng | Field-level accuracy | ~99% tài liệu cấu trúc; thấp hơn cho viết tay/scan kém | Tách theo loại tài liệu/vendor |
| Chất lượng | First-pass yield | Chốt baseline rồi cải thiện | |
| Chất lượng | Lỗi / 1.000 tài liệu | Ngưỡng client chấp nhận | |
| Tốc độ | Cycle time (trọn vòng) | So sánh trước/sau thủ công | ROI chính nằm ở đây |
| Vận hành | Tỷ lệ vào hàng đợi ngoại lệ | Theo dõi để định biên reviewer | |
| Chi phí | Cost/page (OCR vs VLM) | Giám sát %VLM | |

> Lưu ý hiệu chỉnh kỳ vọng: "99% accuracy" áp cho **tài liệu có cấu trúc**. Viết tay, chữ ký kiểu cách, scan < 200 DPI → độ chính xác giảm mạnh và **phải** đẩy HITL — cần truyền đạt rõ với client để tránh kỳ vọng sai.

---

## 6. Quyết định nền tảng: build-vs-buy & cảnh báo "agentic"

**Build-vs-buy theo lớp** (kinh nghiệm: đừng tự xây phần commodity):

| Lớp | Khuyến nghị | Lý do |
|---|---|---|
| OCR cơ bản, PII detection | **Buy/API** (Textract, Google Document AI, Azure DI, Presidio) | Commodity, rẻ, chín; tự xây tốn thời gian không tạo khác biệt |
| Trích xuất field + orchestration + HITL + integration | **Build** | Đây là phần khác biệt & dính nghiệp vụ client |
| VLM | **Buy qua gateway** (Tier 1/2) hoặc **self-host** (Tier 0/3/4) | Theo tier bảo mật `workflow §8.2` |

**Cảnh báo agentic (Gartner):** >40% dự án agentic bị hủy tới 2027; nhiều use case không thật sự cần agentic.
→ **Quyết định:** Giai đoạn 1–2 chạy **luồng tất định + full HITL** (đã đúng `workflow.md`). Chỉ thêm năng lực agentic (vòng tự sửa, tool-calling) ở Giai đoạn 4, **chỉ khi** một use case cụ thể chứng minh ROI mà luồng tất định không đạt. Không gắn nhãn "agentic" để bán.

---

## 7. Lộ trình triển khai theo giai đoạn

Khớp với `workflow §13` (kiến trúc) + `governance §8` (dữ liệu) + use case ROI.

| GĐ | Mục tiêu | Hạng mục chính | Tiêu chí ra |
|---|---|---|---|
| **0 — Discovery** | Chốt scope, KPI, mẫu | Tài liệu mẫu; KPI ký nhận; xác nhận tier tuân thủ; endpoint hạ nguồn | KPI baseline + scope ký |
| **1 — Lõi tất định (1 use case)** | Hóa đơn/AP chạy thật | Ingestion, OCR+LLM, validation, **full HITL + UI**, **integration ERP**, governance mức dự án, **KPI dashboard** | Đạt KPI nghiệm thu use case 1 |
| **2 — Mở rộng năng lực + use case 2** | KYC + tài liệu khó | VLM + phân đoạn vùng + sub-extractor; **exception-queue ops**; schema management; vẫn full HITL | Use case 2 đạt KPI; golden data tích lũy |
| **3 — Nới confidence-gating + scale** | Giảm tải HITL | Dùng golden data nới full-HITL → confidence-gating cho phần đã chứng minh; observability/drift đầy đủ | STP tăng, %HITL giảm, không regression |
| **4 — Agentic (tùy chọn)** | Chỉ nơi ROI rõ | Workflow engine, vòng tự sửa, tool-calling least-privilege | ROI chứng minh vượt luồng tất định |

> Giai đoạn 0 thường bị bỏ qua nhưng là nơi quyết định thành/bại ROI. **Không bắt đầu build khi chưa có KPI ký nhận và tài liệu mẫu.**

---

## 8. Phụ thuộc & bàn giao (handoff)

**Client cung cấp:** tài liệu mẫu; KPI + ngưỡng chấp nhận; endpoint API hạ nguồn + credentials; tài khoản cloud/hạ tầng; xác nhận khung tuân thủ; nhân sự reviewer HITL.

**Dự án giao:** hệ IDP chạy được cho use case đã chốt (theo [§3.1](#31-in-scope-dự-án-giao)); công cụ + quy trình HITL; tài liệu vận hành; bàn giao mã + cấu hình.

**Client tự sở hữu sau bàn giao** (xem [§3.2](#32-out-of-scope-clientdoanh-nghiệp-tự-sở-hữu)): chương trình governance tổ chức, machine unlearning, multi-tenancy doanh nghiệp, vận hành production dài hạn, sửa đổi hệ hạ nguồn. **Trách nhiệm compliance cho các phần out-of-scope thuộc về client** — ghi tường minh trong hợp đồng để tránh gánh nghĩa vụ ngầm.

---

## 9. Rủi ro & giảm thiểu

| Rủi ro | Nguồn (thị trường) | Giảm thiểu |
|---|---|---|
| ROI mơ hồ → dự án do dự/hủy | Everest, Gartner | KPI ký nhận GĐ0; bắt đầu 1 use case; đo baseline trước/sau |
| Bỏ quên tích hợp → triệt tiêu tiết kiệm | Build-vs-buy analysis | Integration layer **in-scope GĐ1**, không để sau |
| Thổi phồng agentic → chi phí leo thang | Gartner (>40% hủy) | Luồng tất định trước; agentic chỉ khi ROI rõ |
| Kỳ vọng accuracy sai (viết tay/scan kém) | Báo cáo HITL | Truyền đạt rõ "99% = tài liệu cấu trúc"; HITL cho phần khó |
| Scope creep (thêm use case/ngôn ngữ giữa chừng) | Kinh nghiệm delivery | Scope ký; use case tuần tự; thay đổi qua change-request |
| Trách nhiệm compliance mơ hồ (outsource) | — | [§8](#8-phụ-thuộc--bàn-giao-handoff) handoff tường minh trong hợp đồng |
| Phụ thuộc số liệu thị trường không chắc | Báo cáo (chênh ~1,45–10,57 tỷ USD) | Dùng xu hướng (CAGR, dịch chuyển agentic), không trích một con số tuyệt đối |

---

## Phụ lục A — Tóm tắt dữ liệu thị trường

Chắt lọc từ báo cáo tổng hợp (Gartner, Forrester, Everest, McKinsey, Microsoft, IBM, ABBYY, UiPath…). Số tuyệt đối **chỉ mang tính tham khảo** — các hãng chênh nhau lớn do khác định nghĩa IDP.

- **Khoảng cách thị trường:** ~80–90% dữ liệu mới là phi cấu trúc; chỉ ~18% tổ chức khai thác được → đó là thị trường.
- **Quy mô/tăng trưởng:** thị trường 2025 phổ biến quanh ~3 tỷ USD; nhóm cao tới ~10,57 tỷ USD; **CAGR hai con số (phần lớn 26–33%)** cả thập kỷ.
- **Phân bố:** Bắc Mỹ ~47,6% thị phần (2025); APAC ~17,8%. **BFSI** lớn nhất (~32,7% năm 2026); **y tế/khoa học đời sống** tăng nhanh nhất.
- **Động lực:** xử lý thủ công chiếm 20–30% chi phí vận hành ngành thâm dụng tài liệu; IDP nhanh gấp ~4 lần, giảm tới ~70% trong ngân hàng. McKinsey: ~70% tổ chức đã thí điểm tự động hóa, ~90% định mở rộng trong 2–3 năm.
- **Use case ROI:** hóa đơn/AP, KYC onboarding, claims bảo hiểm. Tuân thủ (Dodd-Frank, SOX, AML) là động lực đặc thù Bắc Mỹ.
- **Kỳ vọng:** accuracy cao + STP đo được; template-free/zero-shot (cải thiện trong 60–90 ngày đầu); tích hợp đầu-cuối; governance/audit; HITL cho ngoại lệ; cloud/hybrid pay-as-you-go; **ROI 3–6 tháng**; tùy biến không cần kỹ năng ML.
- **Xu hướng 2025–2026:** dịch chuyển IDP → **agentic** (Gartner xếp #1), nhưng cảnh báo >40% dự án agentic hủy tới 2027; DSLM (mô hình ngôn ngữ chuyên ngành) lấp khoảng trống LLM tổng quát.

> Các nguồn từ blog nhà cung cấp mang tính giới thiệu sản phẩm; đối chiếu paper/analyst khi cần số liệu benchmark.

---

## Tài liệu liên quan (nội bộ)

- `workflow.md` — Khung kiến trúc hệ IDP (xử lý, bảo mật, messaging, roadmap kiến trúc).
- `governance.md` — Quản trị, lưu trữ & cải tiến dữ liệu vận hành (phạm vi outsource project).
- Tài liệu này — Scope & kế hoạch triển khai: ánh xạ kỳ vọng thị trường → phạm vi giao hàng → công việc phải xử lý.

---
author: lytinh358@gmail.com
date: 2026-06-05
status: draft
agents: idp
summary: Từ báo cáo thị trường về vận hành & quản trị hệ AI (LLMOps 2026) → xác định các phần cần quản trị/vận hành của hệ IDP và hướng xử lý từng phần, ở altitude outsource project
---

# Quản trị & Vận hành runtime (LLMOps) cho hệ IDP

> **Mục đích:** Áp báo cáo thị trường về *cách quản trị hiệu quả một hệ AI* (LLMOps, AI gateway, observability, evaluation gates, reliability/kill-switch, chuẩn NIST/OWASP/EU AI Act/IMDA) vào hệ IDP hiện tại — liệt kê **các phần cần quản trị & vận hành** và **hướng xử lý từng phần**.
> **Phân biệt với `governance.md`:** File này là **quản trị *runtime/vận hành*** (control plane, observability, eval gate, change control, audit). `governance.md` là **quản trị *dữ liệu*** (vòng đời, lưu trữ, cải tiến). Hai file bổ trợ, không trùng — tham chiếu chéo ở các mục liên quan.
> **Altitude:** Dự án **outsource** — giao một hệ IDP có **cơ chế quản trị/vận hành *được cưỡng chế ở runtime*** + runbook bàn giao. KHÔNG nhận **vận hành production dài hạn** (on-call 24/7, SLA vận hành) hay **chương trình AI-governance cấp tổ chức** — phần đó client sở hữu (nhất quán `scope-implementation.md §3.2`).
> **Nguyên tắc cốt lõi của báo cáo:** *Governance thất bại khi nó nằm trong tài liệu thay vì trong cơ chế cưỡng chế.* Responsible AI 2026 được **cưỡng chế ở thời điểm chạy**, không phải trong văn bản chính sách. Mục tiêu là **controlled autonomy** (tự chủ có kiểm soát).
> **Liên kết nội bộ:** Kiến trúc & cơ chế → `workflow.md`; vòng đời dữ liệu → `governance.md`; phạm vi & KPI → `scope-implementation.md`; ràng buộc & trần chi phí → `cost-finops.md`. File này là lớp *Ops/runtime-governance* ngồi trên cả bốn.

---

## Mục lục

1. Tóm tắt điều hành
2. Đổi tư duy: vận hành một "compound system", không phải một model
3. Các phần cần quản trị & vận hành (bảng chủ — trả lời trực tiếp)
4. Control plane: AI Gateway
5. Observability cấp span
6. Evaluation gates trong CI/CD
7. Reliability & change control: phanh, rollback, kill switch
8. Cost enforcement ở runtime
9. Registry tài sản + audit log + ánh xạ chuẩn
10. Lộ trình vận hành theo giai đoạn
11. Phân định in-scope / out-of-scope & bàn giao
12. Rủi ro vận hành & giảm thiểu
- Phụ lục A — Sơ đồ kiến trúc tham chiếu vận hành
- Phụ lục B — SRE runbook checklist (document-intelligence agent)
- Phụ lục C — Tóm tắt dữ liệu thị trường

---

## 1. Tóm tắt điều hành

Báo cáo chuyển trọng tâm từ *tài chính* sang *vận hành*, với một thông điệp xuyên suốt: **quản trị phải được thực thi ở runtime, không nằm trong tài liệu.** Áp vào hệ IDP, mô hình vận hành hội tụ về:

1. **Một control plane = AI Gateway** (routing + rate limit + budget + PII + guardrail) — đã có ở `workflow.md §8.5`; cần *kích hoạt đầy đủ vai trò control plane*, không chỉ là proxy gọi model.
2. **Bọc bởi observability cấp span** (trace request xuyên prompt→retrieval→tool→guardrail, correlation ID, cost/token/latency, cảnh báo *trước* khi chạm người dùng) — mở rộng `workflow §10` + `governance §6`.
3. **Chặn đầu vào bằng evaluation gates trong CI/CD** (chấm đồng thời *quality + cost + policy*; ngưỡng hallucination/drift/PII trước khi release) — **gap phải xây**, nuôi bằng golden data từ full-HITL.
4. **Bảo vệ bằng least-privilege + change control có rollback + kill switch** — ở giai đoạn đầu, **full-HITL chính là cơ chế "controlled autonomy"** (con người là guardian); guardian-agent là khái niệm Giai đoạn 4/doanh nghiệp.
5. **Neo vào chuẩn** (OWASP LLM Top 10, OWASP Agentic Top 10 12/2025, NIST AI RMF, EU AI Act Art.15, Singapore IMDA 1/2026) với **registry tài sản + audit log bất biến** — đã có nền ở `workflow §8.10, §11`.

> **"Good looks like":** Gartner — >40% dự án agentic bị hủy đến cuối 2027 do *thất bại governance*, không phải công nghệ; nhóm đầu tư guardrail sớm mới đạt quy mô production. Thành công 2026 đo bằng *khả năng kiểm soát & quản trị*, không bằng *triển khai bao nhiêu AI*.

---

## 2. Đổi tư duy: vận hành một "compound system", không phải một model

Một ứng dụng LLM production không phải một model đơn lẻ mà là **hệ thống nhiều thành phần tương tác**: retrieval, routing, nhiều model kích cỡ khác nhau, guardrail lọc input/output, cache, và vòng feedback. Thành công **không do chọn model quyết định**, mà do LLMOps — kiến trúc, governance, observability, kiểm soát chi phí — hoạt động *cùng nhau như một hệ thống*.

Hệ IDP **đã được thiết kế như một compound system** (`workflow.md`): dispatch theo loại vùng (§4.4), OCR + VLM + sub-extractor (§4.5–4.7), Gateway (§8.5), HITL (§4.10), cache (§6.6/6.8), golden-data feedback (`governance §7`). Việc còn lại của tài liệu này: **chốt các điểm phải *quản trị & vận hành* xuyên hệ đó, và hướng xử lý.**

---

## 3. Các phần cần quản trị & vận hành (bảng chủ)

Đây là **trả lời trực tiếp** câu hỏi: các phần cần quản trị/vận hành, trạng thái thiết kế hiện tại, hướng xử lý, và phân định scope. Chi tiết hướng xử lý ở các mục §4–§9.

| # | Phần cần quản trị/vận hành | Trạng thái thiết kế hiện tại | Hướng xử lý (tóm tắt) | Scope |
|---|---|---|---|---|
| **G1** | **Control plane (AI Gateway)** | Có `workflow §8.5` (policy chung) | Kích hoạt đủ vai trò: routing + rate limit + virtual key + budget + PII + guardrail; điểm cưỡng chế duy nhất ([§4](#4-control-plane-ai-gateway)) | **In** |
| **G2** | **Observability cấp span** | Một phần: metrics + tracing `workflow §10`; drift `governance §6` | Trace xuyên prompt→retrieval→tool→guardrail; correlation ID; alert *trước* suy giảm ([§5](#5-observability-cấp-span)) | **In** (công cụ) / **Out** (giám sát dài hạn) |
| **G3** | **Evaluation gates (CI/CD)** | **Gap** — chưa có eval harness | Eval-first: chấm quality+cost+policy; ngưỡng hallucination/drift/PII; nuôi bằng golden data ([§6](#6-evaluation-gates-trong-cicd)) | **In** |
| **G4** | **Versioning & rollback** | Có nền: saga/checkpoint `workflow §6.7/6.8`; pipeline/model version `governance §4.1` | Version prompt/model/schema + đường rollback tự động ([§7](#7-reliability--change-control-phanh-rollback-kill-switch)) | **In** |
| **G5** | **Change control (phanh/công tắc ngắt)** | Một phần: orchestration §4.12, cấm auto-exec tool §8.6 | Pre-check trước hành động; phê duyệt thay đổi nhạy cảm; kill switch; rollback khi vi phạm điều kiện ([§7](#7-reliability--change-control-phanh-rollback-kill-switch)) | **In** |
| **G6** | **Least-privilege / Least-Agency** | Có: Zero Trust `workflow §8.3`, LLM06 §8.6/§11 | Mỗi worker/agent nhận quyền & mức tự chủ tối thiểu; full-HITL là "guardian" GĐ đầu ([§7](#7-reliability--change-control-phanh-rollback-kill-switch)) | **In** |
| **G7** | **Cost enforcement runtime** | Có thiết kế: `cost-finops §8` (3 lớp + Gateway) | Trần cưỡng chế *trước* lời gọi; routing/right-sizing trong pipeline (GĐ3+) ([§8](#8-cost-enforcement-ở-runtime)) | **In** |
| **G8** | **Guardrails an toàn nội dung & bảo mật** | Có: §8.5 (4 nhóm), §8.6 prompt injection, §8.7 toàn vẹn output | Vận hành guardrail như runtime control, không cấu hình tĩnh; gắn eval red-team ([§4](#4-control-plane-ai-gateway), [§6](#6-evaluation-gates-trong-cicd)) | **In** |
| **G9** | **Registry tài sản + audit log** | Có nền: audit redact-by-design §8.10; lineage `governance §4.4` | Registry model/agent (owner, rủi ro, phụ thuộc, phạm vi pháp lý); audit log bất biến WORM ([§9](#9-registry-tài-sản--audit-log--ánh-xạ-chuẩn)) | **In** |
| **G10** | **Ánh xạ chuẩn pháp lý** | Có: OWASP/NIST §8.10, §11; tuân thủ §8.1 | Ánh xạ gateway+guardrail → OWASP LLM/Agentic, NIST AI RMF, EU AI Act Art.15, IMDA ([§9](#9-registry-tài-sản--audit-log--ánh-xạ-chuẩn)) | **In** (cho hệ) / **Out** (chương trình tổ chức) |
| **G11** | **Reliability vận hành (SLO/queue/DLQ)** | Có: NFR §10, DLQ/backpressure §6.4, hai lane §6.5 | SLO + alert + DLQ runbook; bàn giao runbook ([Phụ lục B](#phụ-lục-b--sre-runbook-checklist-document-intelligence-agent)) | **In** (cơ chế + runbook) / **Out** (on-call dài hạn) |
| **G12** | **Guardian agents (agent giám sát agent)** | Không (chưa cần) | Khái niệm doanh nghiệp/GĐ4; GĐ đầu **full-HITL thay thế vai trò guardian** ([§7](#7-reliability--change-control-phanh-rollback-kill-switch)) | **Out** (GĐ đầu) |

> **Đọc bảng:** Phần *bảo mật, audit, control-plane nền* đã được thiết kế tốt. Hai việc phải *xây mới* ở altitude dự án là **G3 (evaluation gates/harness)** và **kích hoạt đầy đủ G1 (Gateway như control plane: virtual key + budget enforcement)** + **G4/G5 (versioning + change control + kill switch)**. Phần còn lại chủ yếu là *vận hành đúng cái đã có* + bàn giao runbook.

---

## 4. Control plane: AI Gateway

**Pattern hội tụ nhất của thị trường:** thay vì cài kiểm soát rải rác từng service, **mọi lời gọi model đi qua một gateway thực thi chính sách** (gateway-plus-guardrails). Hệ IDP đã đặt Gateway ở `workflow §8.5` (ADR-5) — cần kích hoạt đủ vai trò control plane:

| Chức năng gateway | Hướng xử lý cho IDP | Neo |
|---|---|---|
| Định tuyến provider | Định tuyến OCR/VLM provider theo tier (chặn VLM-API ngoài cho Tier 0/3/4) | `workflow §4.4, §8.2` |
| Rate limiting / 429 backoff | Xử lý nút thắt VLM bằng rate-limit tập trung, không hạ cấp chất lượng | `workflow §6.6` |
| Lọc prompt injection + che/ẩn PII | Guardrail input/output; tài liệu = dữ liệu, không phải lệnh | `workflow §8.5, §8.6`, P9 |
| Virtual keys + RBAC | Khóa ảo theo use case/tenant; RBAC; gom audit về một chỗ | `workflow §8.3` |
| **Budget enforcement** | **Cưỡng chế trần chi phí *trước* lời gọi kế tiếp** (per-action/doc/tenant) | `cost-finops §8` |

> **Lợi ích đo được (số thị trường, tham khảo):** tập trung governance + FinOps giảm 20–30% chi phí AI/tháng (cắt lời gọi thừa); autoscaling + edge giảm tới 40% độ trễ p95 lúc cao điểm. Với IDP, giá trị lớn nhất là **giảm bề mặt tấn công + đơn giản hóa audit** (một policy cho mọi provider; đổi model giữ nguyên posture — `§8.5`).

**Nguyên tắc vận hành:** guardrail hoạt động tốt nhất khi *gói chung* với virtual key + budget + rate limit + RBAC trong gateway — không tách rời. Đây là điểm cưỡng chế **duy nhất** của các phần G1/G7/G8/G10.

---

## 5. Observability cấp span

Observability cho LLM khác monitoring truyền thống. Tiêu chuẩn 2026 áp cho IDP:

| Năng lực | Hướng xử lý cho IDP | Trạng thái |
|---|---|---|
| **Distributed tracing cấp span** | Trace xuyên ingestion→PII firewall→dispatch→OCR/VLM/sub-extractor→merge→validation→HITL→integration; **correlation ID per event** liên kết phiên→từng thao tác | Có nền `workflow §8.10, §10` — *mở rộng span cho từng chặng* |
| **Metrics & alert real-time** | Dashboard: phân bố độ trễ, tỷ lệ lỗi, token usage, **cache hit rate**, cost; **alert kích hoạt TRƯỚC khi suy giảm chạm người dùng** | Một phần `§10` — *thêm alert sớm* |
| **Giám sát chất lượng liên tục** | Luồng prompt, **chất lượng retrieval (RAG)**, mẫu hallucination, đỉnh độ trễ, token theo thời gian thực | Nối `governance §6` (drift/anomaly) |
| **Quy mô + chủ quyền dữ liệu** | Trace khối lượng lớn; triển khai SaaS/on-prem/air-gapped theo tier (Tier 0/3/4 on-prem) | `workflow §8.2, §9` |

> **Tích hợp quan trọng:** observability **chia sẻ metadata với lineage** (`governance §4.4`) và **event schema mang cost telemetry** (`governance §4.1` + Phụ lục A: `cost_tokens`, `latency_ms`). Khi alert nổ, ngữ cảnh lineage + cost có sẵn ngay — không phải truy ngược thủ công. Đây là cách *một* hệ telemetry phục vụ cả G2 (ops), cost KPI (`cost-finops §9`) và drift (`governance §6`).

**Ranh giới altitude:** xây *công cụ* observability (dashboard, trace, alert) = **in-scope**; *giám sát/điều tra dài hạn 24/7* = **out-of-scope** trừ hợp đồng bảo trì.

---

## 6. Evaluation gates trong CI/CD

Đây là chỗ tách "demo" khỏi "production" — và là **gap lớn nhất** phải xây. Triết lý *evaluation-first*: kiểm thử có hệ thống là **nền móng**, không phải tính năng đính kèm. Lý do: thay đổi nhỏ ở prompt/retrieval/model version có thể tác động lớn tới output; sự cố production thường truy về thay đổi *chưa được test*.

**Hướng xử lý — đưa vào pipeline:**

| Gate | Nội dung | Neo / nguồn dữ liệu |
|---|---|---|
| **Eval set từ golden data** | Bộ đánh giá lấy từ **xác nhận HITL** (full-HITL → golden data) — đây là tài sản eval *miễn phí* của giai đoạn đầu | `workflow §4.10`, `governance §7` |
| **Pre-production policy gates** | Ngưỡng chấp nhận cho **hallucination, model drift, tuân thủ** trước khi release; mỗi lần chạy chấm điểm *quality + cost + policy* | `cost-finops §9`, `governance §9` |
| **Kiểm soát nhúng CI/CD** | Lineage dữ liệu, **quét PII**, model card, kiểm tra bảo mật/quyền riêng tư, **red-team gate** | `governance §4.4, §5`; `workflow §8.6` |
| **Versioning + rollback** | Version prompt/model/schema; rollback đảm bảo độ tin cậy | [§7](#7-reliability--change-control-phanh-rollback-kill-switch), G4 |

> **Quy tắc "đèn đỏ" gắn chi phí:** eval chấm *cả chi phí*, không chỉ chất lượng — một thay đổi tăng chính xác 1% nhưng tăng chi phí 2× **bị chặn ngay tại gate**. Đây là hiện thực runtime của đường Pareto chính xác–chi phí ở `cost-finops §5`: gate cưỡng chế "chọn ngưỡng chính xác theo rủi ro rồi tối thiểu chi phí", không để chính xác trôi vô tội vạ.

> **Tích hợp với ngưỡng theo use case:** gate dùng *ngưỡng chính xác đã ký ở Giai đoạn 0* (`cost-finops §5.2`: KYC ~99,5% vs nội bộ ~92%) làm tiêu chí pass/fail — mỗi use case một ngưỡng.

---

## 7. Reliability & change control: phanh, rollback, kill switch

Rủi ro agentic khác biệt vì agent **hành động**, không chỉ nói. Phần lớn sự cố tự động hóa xảy ra *trong lúc thay đổi* (config sai, update vội, rollback chưa từng test). **Nguyên tắc cứng:** *nếu một agent không thể bị ràng buộc, nó không nên được trao quyền production.*

**Mẫu an toàn — hướng xử lý cho IDP:**

| Cơ chế | Hướng xử lý | Neo |
|---|---|---|
| **Pre-check trước hành động** | Trước khi đẩy output xuống hạ nguồn (integration), kiểm tra completeness + validation + (eKYC) completeness policy | `workflow §4.8, §6.7` |
| **Peer review / phê duyệt** | Thay đổi nhạy cảm (prompt, schema, ngưỡng confidence) qua review + change-request | `scope §9` (scope creep control) |
| **Cửa sổ thực thi có canh giữ** | Rate limit + backpressure khi tải cao | `workflow §6.4, §6.6` |
| **Rollback tự động** | Saga mức tài liệu + checkpoint/resumable; idempotency state machine (không đóng băng lỗi transient) | `workflow §6.7, §6.8` (ADR-12/13) |
| **Kill switch** | Công tắc ngắt tại Gateway: dừng đường VLM/provider hoặc cả pipeline khi vi phạm điều kiện (cost trần, anomaly, sự cố provider) | `workflow §8.5`; `cost-finops §8.2` |

**Hai cơ chế đang thành chuẩn — và cách áp ở altitude dự án:**

- **Least-privilege / Least-Agency (OWASP):** mỗi worker/agent khởi đầu với **tập quyền & mức tự chủ tối thiểu**. Hệ IDP đã theo: File Proxy giữ credentials, worker chỉ có access_key giới hạn (`§8.3`); cấm auto-execute tool từ nội dung tài liệu (`§8.6`, LLM06). **Giữ nguyên — không nới quyền sớm.**
- **Guardian agents (agent giám sát agent):** Gartner dự báo 40% CIO yêu cầu guardian agent vào 2028. **Ở giai đoạn đầu, full-HITL *chính là* cơ chế guardian** (ADR-11: con người duyệt 100%, bắt hallucination + output bị injection thao túng). Guardian-agent *tự động* là khái niệm Giai đoạn 4/doanh nghiệp — **out-of-scope GĐ đầu**, chỉ cân nhắc khi nới khỏi full-HITL.

> **Đây là điểm "controlled autonomy" cho dự án:** không trao tự chủ rồi giám sát; mà giữ con người làm guardian (full-HITL) cho tới khi golden data + eval gate chứng minh đủ tin để nới — đồng bộ lộ trình `workflow §13` GĐ3.

---

## 8. Cost enforcement ở runtime

Phần chi tiết ở `cost-finops.md` — ở đây chỉ chốt khía cạnh *vận hành* (cưỡng chế, không chỉ theo dõi):

- **Trần ngân sách nhiều lớp cưỡng chế tại Gateway** (per-action / per-document / per-tenant) — *trước* lời gọi kế tiếp (`cost-finops §8`).
- **Routing & right-sizing model trong pipeline** — model nhỏ chuyên biệt cho việc thô. **Lưu ý ADR-10:** routing *động theo độ khó* bị hoãn tới **Giai đoạn 3+** để giữ tính tất định; GĐ1–2 chỉ dùng dispatch *tĩnh* theo loại vùng (đã rẻ đúng-công-cụ). Chi tiết hòa giải: `cost-finops §6.3`.
- **Eval chấm cả cost** (xem [§6](#6-evaluation-gates-trong-cicd)) — chặn thay đổi "tăng nhẹ chính xác, tăng mạnh chi phí".
- **Quy chiếu chi phí về kết quả nghiệp vụ** — cost-per-document, không cost-per-token (`cost-finops §3` C1).

> Báo cáo liệt **cost governance** vào *bảy quyết định guardrail cốt lõi* khi triển khai agent — củng cố việc cost enforcement là phần *vận hành*, không tách rời governance.

---

## 9. Registry tài sản + audit log + ánh xạ chuẩn

Đội Ops không tự nghĩ ra khung mà **ánh xạ vào chuẩn để biện hộ được trước auditor**.

**Hướng xử lý:**

- **Registry tài sản tập trung:** theo dõi mọi model/agent/extractor kèm **owner, phân loại rủi ro, phụ thuộc dữ liệu, phạm vi pháp lý**. Với IDP: mỗi extractor (OCR/VLM/sub-extractor/LLM-field) + version (`model_version`, `pipeline_version` ở `governance §4.1`) vào registry. *Xây mới nhẹ.*
- **Audit log bất biến (WORM):** ánh xạ tới kiểm soát nội bộ + yêu cầu pháp lý. Đã có: audit zone WORM ≥1 năm (`workflow §7.3`), redact-by-design (`§8.10`).
- **Ánh xạ chuẩn:** pattern gateway-plus-guardrails ánh xạ trực tiếp:

| Chuẩn | Điểm neo trong hệ |
|---|---|
| **OWASP LLM Top 10 (2025)** | `workflow §11` (LLM01/02/06/08…) |
| **OWASP Agentic Top 10 (12/2025)** | `workflow §8.10`; áp khi vào GĐ4 agentic |
| **NIST AI RMF (Measure 2.6)** | Observability + eval gate + audit ([§5](#5-observability-cấp-span), [§6](#6-evaluation-gates-trong-cicd)) |
| **EU AI Act Điều 15** (hệ AI rủi ro cao) | Accuracy/robustness/cybersecurity — gateway + eval gate + tier model |
| **Singapore IMDA** (khung agentic đầu tiên, 1/2026) | Tham chiếu khi bật agentic GĐ4 |
| ISO 27001 / PCI DSS / GDPR-PDPA / SBV | `workflow §8.1` |

> **Ranh giới altitude:** ánh xạ chuẩn *cho hệ IDP giao* = in-scope; **chương trình responsible-AI/registry cấp toàn tổ chức** = out-of-scope (client), nhất quán `scope §3.2`.

---

## 10. Lộ trình vận hành theo giai đoạn

Khớp `workflow §13` / `scope §7` / `cost-finops §10`.

| GĐ | Trọng tâm vận hành | Việc làm | Tiêu chí ra |
|---|---|---|---|
| **0 — Discovery** | Chốt SLO + ngưỡng eval | Định nghĩa SLO (độ trễ/lane, uptime); ngưỡng hallucination/drift/PII; ngưỡng chính xác & trần chi phí (ký với client) | SLO + ngưỡng gate được ký |
| **1 — Lõi tất định + full HITL** | Control plane + observability nền | Kích hoạt **Gateway đủ vai trò** (G1); tracing cấp span + alert sớm (G2); audit WORM + registry nền (G9); **full-HITL = guardian** | Gateway/observability/audit chạy; trace xuyên pipeline |
| **2 — Hybrid VLM + use case 2** | Eval gates + change control | **Eval harness** chấm quality+cost+policy (G3); versioning + rollback (G4); kill switch tại Gateway (G5); red-team gate | Mọi release qua eval gate; rollback test được |
| **3 — Golden data → nới gating** | Tự chủ có kiểm soát | Nới full-HITL → confidence-gating dựa golden data + eval; **mở model routing động** (cost-finops §6.3); drift/anomaly đầy đủ | %HITL giảm, không regression chất lượng/chi phí |
| **4 — Agentic (tùy chọn)** | Ràng buộc agent | Least-Agency chặt; (cân nhắc) guardian-agent; OWASP Agentic + IMDA; **kill switch + trần phiên bắt buộc** | Agent *ràng buộc được* trước khi trao quyền production |

> **Bài học "good looks like":** đầu tư guardrail **sớm** (GĐ1–2) là nhóm đạt quy mô production. Đừng để eval gate/change control sang sau — đó là nơi dự án agentic "chết" (Gartner >40% hủy).

---

## 11. Phân định in-scope / out-of-scope & bàn giao

**In-scope (dự án giao — *cơ chế cưỡng chế runtime*):**
- Gateway control plane đầy đủ (routing/rate-limit/virtual-key/budget/PII/guardrail).
- Observability cấp span + alert + dashboard (công cụ).
- Evaluation harness + policy gates trong CI/CD.
- Versioning + rollback + kill switch + least-privilege.
- Registry tài sản nền + audit log WORM + ánh xạ chuẩn cho *hệ*.
- **SRE runbook bàn giao** (SLO, alert, cost-per-document threshold, rollback) — [Phụ lục B](#phụ-lục-b--sre-runbook-checklist-document-intelligence-agent).

**Out-of-scope (client sở hữu sau bàn giao):**
- **Vận hành production dài hạn** (on-call 24/7, SLA vận hành) — trừ hợp đồng bảo trì (`scope §3.2`).
- **Chương trình AI-governance cấp tổ chức** (responsible-AI board, registry toàn doanh nghiệp, RACI).
- **Guardian-agent fleet** quy mô doanh nghiệp.
- Vận hành/điều tra observability dài hạn.

> Trách nhiệm compliance cho phần out-of-scope thuộc client — ghi tường minh trong hợp đồng (nhất quán `scope §8`).

---

## 12. Rủi ro vận hành & giảm thiểu

| Rủi ro | Nguồn (thị trường) | Giảm thiểu |
|---|---|---|
| Governance "nằm trong tài liệu" không cưỡng chế | Báo cáo (nguyên tắc cốt lõi) | Cưỡng chế ở runtime: Gateway + eval gate + kill switch ([§4](#4-control-plane-ai-gateway), [§6](#6-evaluation-gates-trong-cicd), [§7](#7-reliability--change-control-phanh-rollback-kill-switch)) |
| Sự cố trong lúc thay đổi (config/update/rollback) | Báo cáo reliability | Change control: pre-check + review + rollback tự động test được (G4/G5) |
| Thay đổi prompt/model chưa test phá chất lượng | Eval-first | Eval gates trong CI/CD; version + rollback (G3/G4) |
| Trao quyền agent không ràng buộc được | OWASP Least-Agency | Least-privilege; full-HITL làm guardian GĐ đầu; agentic chỉ GĐ4 (G6/G12) |
| Routing động sớm phá tính audit eKYC | ADR-10 | Dispatch tĩnh GĐ1–2; routing động GĐ3 (`cost-finops §6.3`) |
| Không nhìn được hệ → không vận hành được | Báo cáo observability | Tracing cấp span + alert sớm; chia metadata với lineage/cost (G2) |
| Không biện hộ được trước auditor | Báo cáo chuẩn | Registry + audit WORM + ánh xạ OWASP/NIST/EU AI Act/IMDA (G9/G10) |
| Lẫn altitude: ôm vận hành/governance tổ chức | Memory dự án + `scope §3.2` | Giao *cơ chế + runbook*; vận hành dài hạn & chương trình tổ chức là của client ([§11](#11-phân-định-in-scope--out-of-scope--bàn-giao)) |
| Số liệu thị trường (vendor) | Báo cáo | Dùng *pattern & nguyên tắc*, không neo con số tuyệt đối |

---

## Phụ lục A — Sơ đồ kiến trúc tham chiếu vận hành

Luồng control: request → gateway → guardrails/eval → model router → observability → audit. Đây là *góc nhìn vận hành* chồng lên pipeline xử lý ở `workflow §3.1`.

```mermaid
flowchart TD
    REQ["Request / tài liệu vào"] --> GW

    subgraph CP["Control plane"]
        GW["AI Gateway<br/>routing · rate-limit · virtual-key · budget · PII · guardrail"]
        GW --> GRD{"Guardrails + policy gate<br/>(content safety · injection · PII · cost trần)"}
        GRD -->|"vi phạm trần/anomaly"| KILL["Kill switch / chặn lời gọi"]
        GRD -->|"pass"| RTR["Model router<br/>(GĐ1-2: dispatch tĩnh theo loại vùng;<br/>GĐ3+: routing động theo độ khó)"]
    end

    RTR --> EXE["Thực thi: OCR / VLM / sub-extractor / LLM-field"]
    EXE --> HITL["Full-HITL (guardian, GĐ đầu)"]
    HITL --> OUT["Output → ERP/CRM/RAG"]

    EXE -. "span trace + cost/token/latency" .-> OBS[("Observability<br/>tracing cấp span · alert sớm")]
    GW -. .-> OBS
    HITL -. .-> OBS
    OBS -. "chia metadata" .-> LIN[("Lineage — governance.md")]
    GW -. "log bất biến" .-> AUD[("Audit log WORM<br/>↔ OWASP/NIST/EU AI Act/IMDA")]
    HITL -. "golden data" .-> EVAL[("Eval set → CI/CD gate<br/>quality + cost + policy")]
    EVAL -. "gate pass/fail" .-> RTR
```

---

## Phụ lục B — SRE runbook checklist (document-intelligence agent)

Bàn giao cho client; điền số thật ở Giai đoạn 0.

**SLO (theo lane — `workflow §6.5`):**
- [ ] Priority lane (eKYC): độ trễ p95 < _<điền>_; uptime mục tiêu _<điền>_.
- [ ] Bulk lane: throughput _<điền>_ tài liệu/giờ; eventual.
- [ ] Eval real-time (nếu bật): chấm chất lượng < 200ms (chuẩn thị trường).

**Ngưỡng & cost (chốt với client):**
- [ ] Ngưỡng chính xác theo use case (KYC ~99,5% / AP ~99% / nội bộ ~92%) — `cost-finops §5.2`.
- [ ] **Cost-per-document trần** (per-document/per-tenant) cưỡng chế tại Gateway — `cost-finops §8`.
- [ ] Ngưỡng eval gate: hallucination, drift, PII-leak = _<điền>_.

**Alert (kích hoạt *trước* khi chạm người dùng):**
- [ ] Độ trễ p95 vượt SLO; tỷ lệ lỗi tăng; cache hit rate giảm.
- [ ] Cost/document vượt biên; chạm trần ngân sách (per-action/doc/tenant).
- [ ] Drift/anomaly (tỷ lệ HITL-correction một vendor tăng đột biến) — `governance §6`.
- [ ] DLQ depth tăng; provider VLM 429/5xx.

**Change control & rollback:**
- [ ] Mọi thay đổi prompt/model/schema qua eval gate + version.
- [ ] Đường rollback test được (saga/checkpoint — `workflow §6.7/6.8`).
- [ ] Kill switch: cách dừng đường VLM/provider hoặc pipeline; điều kiện kích hoạt.

**Bảo mật & audit:**
- [ ] Least-privilege worker; File Proxy giữ credentials (`workflow §8.3`).
- [ ] Audit WORM ≥1 năm; redact-by-design (`workflow §7.3, §8.10`).
- [ ] Registry: owner/rủi ro/phụ thuộc/phạm vi pháp lý từng model/agent.

---

## Phụ lục C — Tóm tắt dữ liệu thị trường

Chắt lọc từ báo cáo (LLMOps/AI-gateway/observability/governance 2026 + analyst). Số vendor đọc như *mục tiêu định hướng*; ưu tiên *pattern & nguyên tắc* hơn con số tuyệt đối.

- **Nguyên tắc nền:** Responsible AI 2026 được **cưỡng chế ở runtime**, không trong văn bản; *governance thất bại khi nằm trong tài liệu thay vì cơ chế cưỡng chế.* Trọng tâm: **controlled autonomy**.
- **Compound system:** ứng dụng LLM production = retrieval + routing + nhiều model + guardrail + cache + feedback; thành công do *LLMOps phối hợp*, không do chọn model.
- **AI Gateway (pattern hội tụ):** routing + rate-limit + PII + guardrail + virtual-key + budget + RBAC gói trong gateway → giảm phức tạp audit + bề mặt tấn công. Lợi ích: **−20–30% chi phí AI/tháng**; **−40% độ trễ p95** cao điểm (autoscale + edge).
- **Observability:** tracing cấp span + correlation ID; alert *trước* suy giảm; trace *hàng triệu tương tác/ngày*, đánh giá chất lượng real-time **<200ms**; SaaS/on-prem/air-gapped theo chủ quyền dữ liệu.
- **Evaluation gates:** eval-first; nhúng lineage + PII scan + model card + red-team trong CI/CD; policy gate đặt ngưỡng hallucination/drift/tuân thủ; mỗi run chấm **quality + cost + policy**; version + rollback.
- **Reliability/change control:** sự cố phần lớn xảy ra *lúc thay đổi*; pre-check + review + cửa sổ canh giữ + rollback tự động; *agent không ràng buộc được thì không trao quyền production*. **Least-Agency (OWASP)**; **Guardian agents** — Gartner: 40% CIO yêu cầu vào 2028.
- **Cost ở Ops:** routing/right-sizing trong pipeline; trần nhiều lớp cưỡng chế *trước* lời gọi; eval chấm cả cost; cost governance ∈ 7 quyết định guardrail cốt lõi; quy chiếu ROI về chỉ số kinh doanh.
- **Chuẩn:** gateway-plus-guardrails ↔ OWASP LLM Top 10, NIST AI RMF (Measure 2.6), EU AI Act Art.15; **OWASP Agentic Top 10 (12/2025)**; **Singapore IMDA** khung agentic đầu tiên (1/2026); registry tài sản + audit log bất biến.
- **"Good looks like":** Gartner — **>40% dự án agentic hủy đến cuối 2027** do *thất bại governance*; nhóm đầu tư guardrail sớm mới đạt scale. Thành công 2026 đo bằng *kiểm soát & quản trị*, không bằng *lượng AI triển khai*.

> Nguồn vendor mang tính giới thiệu sản phẩm; đối chiếu analyst/chuẩn khi cần. Tài liệu ưu tiên xu hướng & cơ chế.

---

## Tài liệu liên quan (nội bộ)

- `workflow.md` — Kiến trúc (Gateway §8.5, observability/NFR §10, threat map §11, audit §8.10, Zero Trust §8.3, saga/idempotency §6.7/6.8, roadmap §13, ADR-5/8/10/11/12/13).
- `governance.md` — Quản trị **dữ liệu** (event schema §4.1, lineage §4.4, observability/drift §6, golden data §7, KPI §9).
- `scope-implementation.md` — Phạm vi & KPI (out-of-scope vận hành dài hạn/governance tổ chức §3.2, observability/KPI §H, handoff §8).
- `cost-finops.md` — Ràng buộc & trần chi phí (guardrail 3 lớp + Gateway §8, cost KPI §9, hòa giải ADR-10 §6.3, Pareto §5).
- Tài liệu này — Quản trị & vận hành **runtime/LLMOps**: các phần cần quản trị/vận hành + hướng xử lý, ở altitude dự án.

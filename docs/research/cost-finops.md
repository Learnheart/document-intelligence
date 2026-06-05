---
author: lytinh358@gmail.com
date: 2026-06-05
status: draft
agents: idp
summary: Từ báo cáo thị trường FinOps/cost-control cho AI → xác định ràng buộc tài chính và cách quản lý chi phí cho hệ IDP (altitude outsource project), gắn với đường đánh đổi chính xác–chi phí
---

# Ràng buộc tài chính & Quản lý chi phí (FinOps) cho hệ IDP

> **Mục đích:** Chuyển báo cáo thị trường về cost-control cho hệ AI 2026 (EY, Gartner, IDC, FinOps Foundation + benchmark học thuật) thành **các ràng buộc tài chính cụ thể** và **cơ chế quản lý chi phí** mà hệ IDP phải có *ngay từ thiết kế*.
> **Altitude:** Tài liệu viết cho một **dự án outsource** — giao một hệ IDP có chi phí *đoán trước được và có trần cưỡng chế* cho 1–3 use case ROI cao. KHÔNG phải dựng một **chương trình FinOps cấp doanh nghiệp** (chargeback đa đơn vị, ngân sách toàn tổ chức) — phần đó client tự sở hữu (nhất quán `scope-implementation.md §3.2`).
> **Luận điểm cốt lõi:** Agent bị khai tử *không phải vì AI đắt*, mà vì **triển khai trước khi mô hình hóa chi phí**, ROI tính theo giả định "chatbot", và **thiếu phanh tài chính**. Hệ "sống sót" coi FinOps và **ngưỡng chính xác theo use case** là yêu cầu thiết kế, không phải vá sau.
> **Liên kết nội bộ:** Kiến trúc & cơ chế chi phí kỹ thuật → `workflow.md`; vòng đời dữ liệu + event schema mang cost telemetry → `governance.md`; phạm vi giao hàng & KPI nghiệm thu → `scope-implementation.md`. Tài liệu này là lớp *FinOps/cost* ngồi trên ba file đó.

---

## Mục lục

1. Tóm tắt điều hành
2. Bối cảnh: dịch chuyển chi phí cố định → biến đổi & vì sao agent "đội giá"
3. Ràng buộc tài chính (financial constraints) phải tuân
4. Yêu cầu thị trường → Yêu cầu thiết kế chi phí (ánh xạ + in/out-of-scope)
5. Đường đánh đổi chính xác–chi phí (Pareto) — mấu chốt
6. Các đòn bẩy chi phí gắn vào kiến trúc IDP
7. Mô hình cost-per-document (unit economics)
8. Cost guardrails: ba lớp + điểm cưỡng chế
9. Đo lường & KPI chi phí
10. Lộ trình chi phí theo giai đoạn
11. Rủi ro chi phí & giảm thiểu
- Phụ lục A — Tóm tắt dữ liệu thị trường
- Phụ lục B — Bảng mô hình cost-per-document (mẫu điền)

---

## 1. Tóm tắt điều hành

Chi phí AI doanh nghiệp 2026 đã dịch từ mô hình **license/lao động cố định** sang **tiêu thụ compute biến đổi** — nơi hóa đơn vendor chỉ phản ánh *một phần* rủi ro tài chính thực. Hệ quả với một hệ IDP:

1. **Đo theo "chi phí mỗi tài liệu xử lý xong", không phải mỗi token.** Đơn vị nghiệp vụ là *cost-per-document* (đã qua validation + HITL + đẩy hạ nguồn), không phải chi phí inference thô. Khớp KPI `cost/page` đã có ở `scope-implementation.md §5`.
2. **Trần chi phí phải *cưỡng chế trước khi tiêu*, không cảnh báo sau khi nhận hóa đơn.** Khi mức tiêu thụ tích lũy của một phiên/ tài liệu vượt ngưỡng → chặn lời gọi model kế tiếp tại **AI Gateway** (`workflow.md §8.5`), không phải đợi báo cáo cuối tháng.
3. **Chính xác và chi phí là một đường đánh đổi (Pareto), không phải hai mục tiêu độc lập.** Ta *chọn ngưỡng chính xác cho từng use case theo rủi ro* rồi tối thiểu hóa chi phí để đạt ngưỡng đó — KYC cần ~99,5%, phân loại nội bộ 92% là đủ. Vài phần trăm chính xác cuối cùng thường đắt gấp đôi.
4. **Phanh tài chính là yêu cầu nghiệm thu, không phải "nice-to-have".** 98% doanh nghiệp đã quản lý chi tiêu AI nhưng chỉ 44% có financial guardrails; 96% báo cáo vượt dự toán. Hệ này phải giao kèm guardrails ba lớp (per-action / per-document / per-tenant).

**Điểm hòa giải với kiến trúc hiện có:** Phần lớn "đòn bẩy chi phí" của thị trường đã có chỗ neo trong `workflow.md` (cache theo hash §6.6, dispatch theo loại vùng tránh VLM đắt §4.4, hai lane + batch API §6.5, giám sát %VLM §10, gateway §8.5). **Một mâu thuẫn phải xử lý tường minh:** đòn bẩy "model tier routing động theo độ khó" mà thị trường khuyến nghị chính là **dynamic model selection đã bị ADR-10 hoãn lại** để giữ tính tất định — nên nó là đòn bẩy **Giai đoạn 3+** (sau golden data), không bật ở Giai đoạn 1–2 (xem [§6](#6-các-đòn-bẩy-chi-phí-gắn-vào-kiến-trúc-idp)).

---

## 2. Bối cảnh: dịch chuyển chi phí cố định → biến đổi & vì sao agent "đội giá"

### 2.1. Cú dịch chuyển nền tảng

Quy trình tuyến tính đơn giản 2023 tốn ~0,04 USD/tương tác; hệ orchestrated 2026 (có tool, suy luận, vòng lặp) lên ~1,20 USD — **cao ~30×** (EY). Vấn đề với CFO: **hóa đơn vendor chỉ là một phần** tổng rủi ro tài chính. FinOps không còn là chuyện cloud — State of FinOps 2026: **98%** nay quản lý chi tiêu AI (từ 31% hai năm trước), "FinOps cho AI" là ưu tiên #1.

### 2.2. Bốn cơ chế khiến chi phí agent leo thang (và dự án bị hủy)

| Cơ chế | Bản chất | Hệ quả cho IDP |
|---|---|---|
| **Hệ số nhân agentic** | Agent gửi lại toàn bộ ngữ cảnh tích lũy mỗi bước → cần 5–30× token/tác vụ; ngữ cảnh gửi lại chiếm tới 62% hóa đơn (Gartner 3/2026) | Vòng tự sửa đa bước (`workflow §4.12`, Giai đoạn 4) là nơi chi phí phình — lý do càng phải hoãn agentic tới khi ROI rõ |
| **Nghịch lý giá giảm – bill tăng** | Giá/token giảm ~80% từ giữa 2023 nhưng tổng chi AI tăng **483%** (2024→2026): tổng chi = đơn giá × khối lượng | Không bao giờ ngân sách theo đơn giá token; ngân sách theo *khối lượng tài liệu × chi phí mỗi tài liệu* |
| **Triển khai trước mô hình chi phí** | Quyết định deploy có trước cost model; ROI ban đầu giả định mức tiêu thụ "chatbot" trong khi thực tế cao hơn một bậc độ lớn | Bắt buộc **mô hình cost-per-document ở Giai đoạn 0** trước khi build ([§7](#7-mô-hình-cost-per-document-unit-economics)) |
| **Thiếu phanh tài chính** | 98% quản lý chi tiêu nhưng chỉ **44%** có financial guardrails; 96% vượt dự toán (IDC); G1000 có thể bị đánh giá thấp tới 30% chi phí hạ tầng AI đến 2027 (IDC FutureScape) | Guardrails cưỡng chế là *điều khoản nghiệm thu*, không phải tùy chọn ([§8](#8-cost-guardrails-ba-lớp--điểm-cưỡng-chế)) |

> **Cảnh báo "kinh điển":** Uber cạn ngân sách công cụ AI coding cả năm 2026 chỉ trong 4 tháng (~3× kế hoạch); một khoản rò rỉ chi tiêu cloud tập thể ~400 triệu USD trong nhóm Fortune 500 do phiên agent chạy *không có trần chi phí mỗi phiên*. Đây là cái Gartner gọi tên khi dự báo **>40% dự án agentic bị hủy tới 2027** — cùng cảnh báo agentic ở `scope-implementation.md §6`.

---

## 3. Ràng buộc tài chính (financial constraints) phải tuân

Đây là phần trả lời trực tiếp: **các ràng buộc cứng** hệ IDP phải thỏa, suy ra từ kỳ vọng FinOps 2026.

| # | Ràng buộc | Diễn giải | Neo thiết kế |
|---|---|---|---|
| **C1** | **Đơn vị chi phí = chi phí mỗi tài liệu xử lý xong** | Đo cost-per-document (sau validation/HITL/đẩy hạ nguồn), không phải cost-per-token; phân tách theo loại tài liệu/vendor/tier | `scope §5` (cost/page), `workflow §10` (%VLM) |
| **C2** | **Trần chi phí cưỡng chế *trước khi tiêu*** | Tích lũy token/phiên hoặc /tài liệu vượt ngưỡng → chặn lời gọi model kế tiếp, không cảnh báo sau hóa đơn | `workflow §8.5` (Gateway), [§8](#8-cost-guardrails-ba-lớp--điểm-cưỡng-chế) |
| **C3** | **Chi phí đoán trước được (predictable, capped)** | Mỗi loại tài liệu có *biên chi phí trên* đã mô hình hóa; không để chi "trôi" theo độ hào hứng người dùng | [§7](#7-mô-hình-cost-per-document-unit-economics), `scope §3.3` (giả định) |
| **C4** | **Ngưỡng chính xác theo rủi ro use case** | Không tối đa hóa chính xác mọi nơi; chốt ngưỡng theo rủi ro rồi tối thiểu chi phí đạt ngưỡng | [§5](#5-đường-đánh-đổi-chính-xác–chi-phí-pareto--mấu-chốt) |
| **C5** | **Bỏ "ngộ nhận mô hình lớn" (Big Model Fallacy)** | Không dùng model frontier cho mọi tác vụ; định tuyến theo độ khó *khi đã đủ điều kiện tất định/golden data* | ADR-1/ADR-10, [§6](#6-các-đòn-bẩy-chi-phí-gắn-vào-kiến-trúc-idp) |
| **C6** | **Mọi khoản chi gắn được metadata** | Liên kết chi phí → tài liệu, use case, model version, tenant để truy vết & quy trách | `governance §4.1` + Phụ lục A (`cost_tokens`, `latency_ms`) |
| **C7** | **Quản trị tài chính liên tục** | Cảnh báo ngân sách, quota, phát hiện bất thường, review hằng tháng, post-mortem vượt chi — vào nhịp vận hành | `governance §6` (observability), [§9](#9-đo-lường--kpi-chi-phí) |
| **C8** | **ROI/payback < 6 tháng với chi phí có trần** | Payback neo <6 tháng (như `scope §1`), nhưng kèm điều kiện chi phí có giới hạn dự đoán được | `scope §5, §7` |

> **Ranh giới altitude (rất quan trọng):** Các ràng buộc trên là về *cost-control của bản thân hệ IDP giao cho client*. **Chương trình FinOps cấp tổ chức** (chargeback/showback đa đơn vị kinh doanh, ngân sách AI toàn doanh nghiệp, danh mục TCO nhiều năm) **thuộc client** — out-of-scope, đã ghi ở `scope-implementation.md §3.2`. Hệ này *cung cấp telemetry & guardrails* để client cắm vào FinOps của họ, nhưng không *là* nền FinOps doanh nghiệp.

---

## 4. Yêu cầu thị trường → Yêu cầu thiết kế chi phí (ánh xạ)

Mỗi kỳ vọng FinOps được dịch thành yêu cầu cụ thể, kèm phân định **in-scope (hệ giao)** vs **out-of-scope (client sở hữu)** — tránh gánh nghĩa vụ ngầm.

| Kỳ vọng thị trường | Yêu cầu cụ thể | In / Out-of-scope | Trạng thái thiết kế |
|---|---|---|---|
| Đo theo kết quả nghiệp vụ | Cost-per-document dashboard, tách theo loại/vendor/tier | **In** | Gap nhỏ — mở rộng KPI dashboard `scope §H` |
| Trần chi phí cưỡng chế trước tiêu | Budget enforcement tại Gateway (per-action/doc/tenant) | **In** | Gap — Gateway có `workflow §8.5` nhưng *cơ chế trần* phải xây |
| Quản trị tài chính liên tục | Alert ngân sách, anomaly chi phí, post-mortem | **In** (công cụ) / **Out** (nhịp vận hành dài hạn) | Tái dùng observability `governance §6` + alert chi phí |
| Gắn metadata mọi khoản chi | `cost_tokens`/`latency_ms`/`tenant_id` trong event | **In** | Có sẵn schema `governance §4.1` + Phụ lục A — chỉ cần *điền & tổng hợp* |
| Bỏ Big Model Fallacy | Model tier routing theo độ khó | **In** nhưng **Giai đoạn 3+** | Hoãn bởi ADR-10 (xem [§6.3](#63-mâu-thuẫn-với-adr-10-và-cách-hòa-giải)) |
| Prompt/semantic caching | Cache system prompt + cache theo hash tài liệu | **In** | Có `workflow §6.6, §6.8` (result cache) |
| Batch cho khối lượng lớn | Batch API cho bulk lane | **In** | Có `workflow §6.5` |
| Chargeback/showback đa BU | Phân bổ chi phí xuyên đơn vị kinh doanh | **Out** (client) | `scope §3.2` |
| Ngân sách AI toàn tổ chức | Quota/forecast cấp doanh nghiệp | **Out** (client) | `scope §3.2` |

> **Đọc bảng:** Bốn việc phải *xây/điền thêm* ở altitude dự án là: **(a) cơ chế trần chi phí cưỡng chế tại Gateway**, **(b) tổng hợp cost telemetry từ event schema**, **(c) cost-per-document dashboard**, **(d) model tier routing — nhưng để Giai đoạn 3+**. Phần caching/batch/dispatch đã có nền.

---

## 5. Đường đánh đổi chính xác–chi phí (Pareto) — mấu chốt

Đây là điểm nối hai câu hỏi (độ chính xác ↔ chi phí). **Không tối đa hóa cả hai** — chọn *ngưỡng chính xác cho từng use case theo rủi ro*, rồi tối thiểu hóa chi phí để đạt ngưỡng đó.

### 5.1. Bằng chứng benchmark (10.000 hồ sơ tài chính SEC 10-K/10-Q/8-K)

| Kiến trúc | F1 | Chi phí (so baseline tuần tự) | Vị trí Pareto |
|---|---|---|---|
| **Reflexive** (tự sửa lặp) | **0,943** (cao nhất) | **2,3×** | Đắt — chỉ xứng khi rủi ro rất cao |
| **Hierarchical** (phân cấp) | 0,921 | 1,4× | **Tốt nhất trên biên Pareto** |
| **Hybrid** (lai) | phục hồi **89%** phần cải thiện của reflexive | **1,15×** | Thực dụng nhất — gần như cùng chính xác, chi phí thấp hơn nhiều |

> **Kết luận thực dụng:** vài phần trăm chính xác cuối cùng thường **đắt gấp đôi**. Cấu hình lai (OCR rẻ trước + chọn lọc dùng model mạnh/HITL cho phần khó) chiếm gần hết phần cải thiện với chi phí thấp — đây đúng triết lý dispatch theo loại vùng + full-HITL của `workflow.md`.

### 5.2. Ngưỡng chính xác theo use case (chốt ở Giai đoạn 0)

| Use case | Ngưỡng chính xác mục tiêu | Cơ chế đạt ngưỡng (rẻ → đắt) | Ghi chú rủi ro |
|---|---|---|---|
| **KYC onboarding** | ~99,5% (tuân thủ) | Full-HITL 100% giai đoạn đầu; giữ HITL lâu nhất | Rủi ro pháp lý/tuân thủ cao → chấp nhận chi phí cao hơn |
| **Hóa đơn / AP** | ~99% tài liệu cấu trúc; STP 60–80% hóa đơn "sạch" | OCR+LLM + confidence-gating *sau golden data* | ROI rõ nhất; nới HITL khi đủ tin |
| **Phân loại / định tuyến nội bộ** | ~92% là đủ | Model nhỏ/OCR, ít hoặc không HITL | Rủi ro thấp → tối thiểu chi phí |

> Ngưỡng phải **client ký nhận ở Giai đoạn 0** cùng KPI (`scope §5`), vì nó quyết định trực tiếp chi phí mỗi tài liệu và lượng HITL.

### 5.3. Tham chiếu vendor (đọc như mục tiêu, không bảo chứng)

Hyperscience mô tả agent định hướng-mục tiêu điều phối tập mô hình chuyên biệt (model riêng, VLM, LLM) với **least-cost routing**, cho phép ưu tiên *chính xác / tốc độ / chi phí / tuân thủ* theo kết quả nghiệp vụ; tuyên bố 99,5% chính xác, 98% tự động hóa, ROI 272%/5 năm, payback <6 tháng. **Đây là số vendor** — dùng làm mục tiêu định hướng, đối chiếu thực tế.

---

## 6. Các đòn bẩy chi phí gắn vào kiến trúc IDP

Từ audit thực tế, bốn đòn bẩy giảm **50–70% chi phí agent trong ~2 tuần**. Dưới đây ánh xạ từng đòn bẩy vào kiến trúc `workflow.md`, và **giai đoạn được phép bật**.

### 6.1. Đòn bẩy có thể bật ngay (Giai đoạn 1–2, không phá vỡ tính tất định)

| Đòn bẩy | Cơ chế | Neo kiến trúc | Tiết kiệm tham khảo |
|---|---|---|---|
| **Dispatch theo loại vùng** | Vùng text → OCR rẻ; chỉ vùng khó (hình/diagram) → VLM đắt; tránh ép cả trang qua VLM | `workflow §4.4`, ADR-9 | Lớn — cốt lõi tránh "VLM cho mọi thứ" |
| **Prompt caching** | Cache system prompt + ngữ cảnh lặp; input đã cache tính chỉ 10–25% chi phí input thường | Gateway `workflow §8.5` | ~2.000 USD/ngày ở quy mô 5.000 vòng/ngày (số thị trường) |
| **Result cache theo hash** | Tài liệu trùng/lặp không chạy lại VLM; chỉ ghi cache khi thành công | `workflow §6.6, §6.8` | Cao với khối lượng lặp |
| **Batch API (bulk lane)** | Số hóa hàng loạt qua batch API rẻ (~ −50%) | `workflow §6.5` | ~50% cho lane bulk |
| **Cắt tỉa cửa sổ ngữ cảnh** | Không gửi lại ngữ cảnh thừa (ngữ cảnh gửi lại = mục tối ưu lớn nhất, tới 62% bill) | Thiết kế prompt extractor `§4.9` | Lớn ở luồng nhiều bước |

### 6.2. Đòn bẩy giảm khối lượng (xuyên suốt)

- **Gate chất lượng đầu vào sớm** (DPI/định dạng) → tài liệu rác không tiêu token VLM, đẩy thẳng hàng đợi ngoại lệ (`scope §4-A`, `workflow §4.1`).
- **Confidence chỉ để ưu tiên HITL, không gate** ở Giai đoạn 1 (`workflow §4.10`) — nhưng *chính golden data từ HITL* mở đường nới confidence-gating sau, giảm khối lượng HITL (chi phí lao động).

### 6.3. Mâu thuẫn với ADR-10 và cách hòa giải

> **Đây là điểm tích hợp quan trọng nhất.** Thị trường khuyến nghị mạnh **"model tier routing theo độ khó"** (Haiku cho việc thô, Opus cho suy luận khó). Nhưng đó **chính là dynamic model selection theo confidence/tải đã bị `workflow.md` ADR-10 loại bỏ ở giai đoạn đầu** để giữ tính tất định (cùng input → cùng đường xử lý, dễ test/audit cho eKYC).

**Hòa giải theo lộ trình hiện có — không sửa ADR-10:**

| Giai đoạn | Đòn bẩy routing được phép | Lý do |
|---|---|---|
| **GĐ 1–2 (tất định + full HITL)** | Chỉ **dispatch *tĩnh* theo loại vùng** (text→OCR, bảng→table extractor, hình→VLM). Đây *đã là* một dạng "tier routing theo loại nội dung", giữ tính tất định | Audit/test được; xóa mâu thuẫn load-aware vs tier accuracy (ADR-10) |
| **GĐ 3+ (sau golden data)** | Mở **model tier routing động theo độ khó/confidence** cho phần đã chứng minh đủ tin — đồng bộ với việc nới full-HITL → confidence-gating | Golden data cho cơ sở chứng minh; đúng `workflow §13` GĐ3 |

> **Diễn giải:** "Bỏ Big Model Fallacy" (C5) ở dự án này được hiện thực **theo hai nhịp**: ngay lập tức bằng *dispatch tĩnh đúng-công-cụ* (đã rẻ, không cần frontier cho text); và *động theo độ khó* chỉ khi đã có golden data để giữ kỷ luật tất định/audit. Không bật routing động sớm để "tiết kiệm" rồi đánh mất khả năng audit của eKYC.

### 6.4. Đòn bẩy thuộc Giai đoạn 4 (agentic — cẩn trọng)

Vòng tự sửa/tool-calling (`workflow §4.12`, `§13` GĐ4) là **nơi hệ số nhân agentic 5–30× đánh vào**. Chỉ bật khi một use case chứng minh ROI vượt luồng tất định, và **bắt buộc kèm trần chi phí mỗi phiên** ([§8](#8-cost-guardrails-ba-lớp--điểm-cưỡng-chế)) — đúng bài học Fortune 500/Uber.

---

## 7. Mô hình cost-per-document (unit economics)

**Bắt buộc ở Giai đoạn 0**, trước khi build (chống bẫy "deploy trước cost model"). Cost-per-document gồm chi phí máy + chi phí lao động HITL:

```
Cost/doc = Σ(cost_OCR) + Σ(cost_VLM × tỷ_lệ_vùng_VLM) + cost_sub_extractors
         + cost_LLM_field_extraction (sau cache)
         + cost_HITL_lao_động (= thời_gian_duyệt × đơn_giá_reviewer × tỷ_lệ_vào_HITL)
         + cost_hạ_tầng_phân_bổ (queue, storage, gateway)
```

Hai biến chi phối lớn nhất:
- **Tỷ lệ vùng đi VLM (%VLM)** — đã là metric NFR (`workflow §10`); kéo giảm bằng dispatch theo loại vùng + cache.
- **Tỷ lệ vào HITL** — ở Giai đoạn 1 là **100%** (full-HITL), nên *chi phí lao động chiếm phần lớn cost-per-document ban đầu*. Đây là đánh đổi có chủ đích (ADR-11): an toàn + sinh golden data. Lộ trình hạ chi phí = nới sang confidence-gating khi đủ golden data (GĐ3), giảm tỷ lệ HITL.

> **Hệ quả ngân sách quan trọng:** ROI Giai đoạn 1 **không** đến từ chi phí máy thấp (vì 100% HITL còn tốn lao động), mà từ **cycle time giảm + nền golden data**. Đừng hứa "tiết kiệm lao động ngay GĐ1" — tiết kiệm thực đến ở GĐ3 khi %HITL giảm. (Truyền đạt rõ với client để tránh kỳ vọng ROI sai — nối `scope §9`.)

Bảng mẫu điền cho 2 use case ở **Phụ lục B**.

---

## 8. Cost guardrails: ba lớp + điểm cưỡng chế

Yêu cầu kỹ thuật mới nổi: **trần cưỡng chế trước khi tiêu** — khi tiêu thụ tích lũy vượt ngưỡng, phiên/ tài liệu bị chấm dứt *TRƯỚC lời gọi LLM kế tiếp* (khác trần cấp tài khoản hay cảnh báo ngân sách vốn chỉ báo *sau* khi đã vượt).

### 8.1. Ba lớp (mỗi lớp bắt loại lỗi lớp khác bỏ sót)

| Lớp | Phạm vi | Ví dụ ngưỡng | Hành vi khi vượt |
|---|---|---|---|
| **Per-action** | Mỗi lời gọi model | Số token/ lời gọi, kích thước ngữ cảnh | Từ chối lời gọi, log anomaly |
| **Per-document / per-session** | Trọn vòng một tài liệu | Tổng token/ tài liệu vượt biên mô hình hóa ([§7](#7-mô-hình-cost-per-document-unit-economics)) | Dừng xử lý tự động → đẩy hàng đợi ngoại lệ + cảnh báo (KHÔNG release dở) |
| **Per-tenant / per-team** | Quota theo khách hàng/đội | Ngân sách ngày/tháng | Throttle/đợi; báo client; không tràn sang tenant khác |

### 8.2. Điểm cưỡng chế = AI Gateway

`workflow.md §8.5` đã đặt **mọi lời gọi model qua một Gateway** áp policy chung. Đây là *điểm tự nhiên duy nhất* để cưỡng chế trần chi phí: Gateway đếm token tích lũy theo correlation/session ID, so ngưỡng **trước** khi forward lời gọi kế tiếp; vượt → chặn + phát event anomaly. Tận dụng sẵn cơ chế 429/backoff/rate-limit ở `§6.6`.

> Lưu ý nhất quán với `workflow §6.8`: chặn-vì-vượt-trần là **kết cục nghiệp vụ tất định** (cache được, hỏi lại vẫn chặn), khác lỗi transient — phân loại đúng để không "đóng băng" nhầm.

### 8.3. Quản trị tài chính liên tục (nhịp vận hành)

- **Alert ngân sách + anomaly chi phí**: ví dụ chi phí/tài liệu của một vendor tăng đột biến (nối drift/anomaly `governance §6`).
- **Review hằng tháng + post-mortem vượt chi** đưa vào runbook.
- **Cập nhật forecast** theo khối lượng thực.

> Phần *công cụ* (alert/dashboard/anomaly) là **in-scope**; *vận hành dài hạn* (on-call FinOps, review định kỳ nhiều năm) là **out-of-scope** trừ hợp đồng bảo trì (nhất quán `scope §3.2`).

---

## 9. Đo lường & KPI chi phí

Mở rộng KPI ở `scope §5` và event schema ở `governance §4.1` (đã có `cost_tokens`, `latency_ms` ở Phụ lục A — chỉ cần *điền & tổng hợp*).

| Nhóm | KPI chi phí | Ý nghĩa | Nguồn dữ liệu |
|---|---|---|---|
| **Unit economics** | **Cost-per-document** (tách loại/vendor/tier) | Đơn vị FinOps cốt lõi (C1) | event `cost_tokens` + cost lao động HITL |
| | Cost máy vs cost lao động (tỷ trọng) | Hiểu ROI đến từ đâu theo giai đoạn | [§7](#7-mô-hình-cost-per-document-unit-economics) |
| **Hiệu suất chi phí** | %VLM (tỷ lệ vùng đi VLM) | Đòn bẩy chi phí lớn nhất phần máy | `workflow §10` |
| | Cache hit rate (prompt + result) | Đo hiệu quả caching | Gateway + result cache |
| | Tỷ lệ vào HITL | Đòn bẩy chi phí lao động; mục tiêu giảm dần | `governance §9` |
| **Guardrail** | Số lần chạm trần (per-action/doc/tenant) | Phát hiện cấu hình/ngưỡng sai sớm | Gateway anomaly |
| | % tài liệu vượt biên chi phí mô hình hóa | Sức khỏe dự báo chi phí | [§7](#7-mô-hình-cost-per-document-unit-economics) |
| **ROI** | Payback (mục tiêu <6 tháng) + cycle time trước/sau | Chứng minh ROI có trần | `scope §5` |

---

## 10. Lộ trình chi phí theo giai đoạn

Khớp lộ trình `workflow §13` / `scope §7`.

| GĐ | Trọng tâm chi phí | Việc làm | Tiêu chí ra |
|---|---|---|---|
| **0 — Discovery** | **Mô hình cost-per-document + ngưỡng chính xác** | Lập unit economics 2 use case; client ký ngưỡng chính xác theo rủi ro; chốt ngân sách & trần | Cost model + ngưỡng + trần được ký |
| **1 — Lõi tất định + full HITL** | Đo & guardrail nền | Cost telemetry vào event schema; cost-per-document dashboard; **guardrail 3 lớp tại Gateway**; caching + batch (bulk) | Dashboard + guardrails chạy; cost/doc đo được |
| **2 — Hybrid VLM + use case 2** | Tối ưu phần máy | Dispatch theo loại vùng giảm %VLM; result cache theo hash; tinh chỉnh prompt giảm ngữ cảnh gửi lại | %VLM & cost/doc trong biên; không regression chính xác |
| **3 — Golden data → nới gating** | **Giảm chi phí lao động** | Nới full-HITL → confidence-gating (giảm %HITL); **mở model tier routing động** theo độ khó | %HITL giảm, cost/doc giảm, giữ ngưỡng chính xác |
| **4 — Agentic (tùy chọn)** | Khống chế hệ số nhân agentic | Chỉ nơi ROI rõ; **bắt buộc trần chi phí mỗi phiên**; giám sát ngữ cảnh gửi lại | ROI vượt luồng tất định *sau* khi trừ chi phí agentic |

> Giai đoạn 0 (lập cost model) thường bị bỏ qua — đây chính là nơi dự án agentic "chết". **Không build khi chưa có cost-per-document model + ngưỡng chính xác + trần chi phí được ký.**

---

## 11. Rủi ro chi phí & giảm thiểu

| Rủi ro | Nguồn (thị trường) | Giảm thiểu |
|---|---|---|
| Deploy trước cost model → vượt chi cấu trúc | Audit FinOps; Gartner | Cost-per-document model **bắt buộc GĐ0** ([§7](#7-mô-hình-cost-per-document-unit-economics)) |
| Hệ số nhân agentic (5–30×) đốt token | Gartner 3/2026 | Hoãn agentic tới GĐ4; trần chi phí mỗi phiên; cắt tỉa ngữ cảnh |
| Thiếu phanh tài chính → bill vượt | IDC (96% vượt); FinOps (chỉ 44% có guardrail) | Guardrail 3 lớp cưỡng chế tại Gateway ([§8](#8-cost-guardrails-ba-lớp--điểm-cưỡng-chế)) |
| Big Model Fallacy (frontier cho mọi việc) | EY/FinOps | Dispatch tĩnh đúng-công-cụ ngay; routing động ở GĐ3 ([§6.3](#63-mâu-thuẫn-với-adr-10-và-cách-hòa-giải)) |
| Kỳ vọng ROI sai (chờ tiết kiệm lao động ngay GĐ1) | — | Truyền đạt: GĐ1 100% HITL → ROI đến từ cycle time + golden data, không phải cost máy ([§7](#7-mô-hình-cost-per-document-unit-economics)) |
| Theo đuổi vài % chính xác cuối → chi phí gấp đôi | Benchmark Pareto | Chốt ngưỡng theo rủi ro use case; cấu hình hybrid ([§5](#5-đường-đánh-đổi-chính-xác–chi-phí-pareto--mấu-chốt)) |
| Routing động sớm phá tính audit của eKYC | ADR-10 | Giữ tất định GĐ1–2; chỉ nới sau golden data |
| Lẫn altitude: ôm FinOps doanh nghiệp | Memory dự án + `scope §3.2` | Giao telemetry+guardrails của *hệ*; chargeback/ngân sách tổ chức là của client |
| Số liệu thị trường không chắc (vendor) | Báo cáo | Dùng *xu hướng & cơ chế*, không neo một con số tuyệt đối (vendor đọc như mục tiêu) |

---

## Phụ lục A — Tóm tắt dữ liệu thị trường

Chắt lọc từ báo cáo tổng hợp (EY, Gartner, IDC, FinOps Foundation + benchmark học thuật). Số tuyệt đối **chỉ tham khảo** — định nghĩa khác nhau giữa các hãng; số vendor đọc như mục tiêu định hướng.

- **Dịch chuyển chi phí:** tuyến tính 2023 ~0,04 USD/tương tác → orchestrated 2026 ~1,20 USD (~30×) — EY. Hóa đơn vendor chỉ phản ánh *một phần* rủi ro tài chính.
- **FinOps cho AI:** State of FinOps 2026 (1.192 người, ~83 tỷ USD chi tiêu/năm): **98%** quản lý chi tiêu AI (từ 31% hai năm trước); "FinOps cho AI" ưu tiên #1; chỉ **44%** có financial guardrails.
- **Cơ chế đội giá:** agentic cần **5–30×** token/tác vụ; ngữ cảnh gửi lại chiếm tới **62%** hóa đơn (Gartner 3/2026). Giá/token giảm ~80% từ giữa 2023 nhưng tổng chi AI tăng **+483%** (2024→2026).
- **Vượt dự toán:** IDC — **96%** doanh nghiệp vượt chi phí AI ban đầu; IDC FutureScape 2026 — G1000 có thể bị đánh giá thấp tới **30%** chi phí hạ tầng AI đến 2027 (workload agentic luôn-bật).
- **Cảnh báo cụ thể:** Uber cạn ngân sách AI coding 2026 trong 4 tháng (~3× kế hoạch); ~400 triệu USD rò rỉ chi tiêu cloud nhóm Fortune 500 do phiên agent không trần/ phiên. Gartner: **>40%** dự án agentic hủy tới 2027.
- **Benchmark Pareto (10.000 hồ sơ SEC):** reflexive F1 0,943 @ 2,3×; hierarchical F1 0,921 @ 1,4× (tốt nhất Pareto); hybrid phục hồi **89%** cải thiện @ **1,15×**.
- **Đòn bẩy giảm 50–70%/2 tuần:** (1) prompt caching (input cache = 10–25% chi phí input); (2) model tier routing; (3) cắt tỉa ngữ cảnh; (4) trần ngân sách 3 lớp. Bổ sung: batch API (~−50%), semantic caching.
- **Yêu cầu doanh nghiệp 2026 (đã cụ thể hóa thành điều khoản nghiệm thu):** đo cost-per-outcome; quản trị tài chính liên tục; trần cưỡng chế trước tiêu; bỏ Big Model Fallacy; payback <6 tháng có trần.
- **Tham chiếu vendor:** Hyperscience — agent điều phối tập model chuyên biệt, least-cost routing, tuyên bố 99,5% chính xác / 98% tự động hóa / ROI 272% (5 năm) / payback <6 tháng (số vendor).

> Nguồn vendor mang tính giới thiệu sản phẩm; đối chiếu analyst/paper khi cần benchmark chính xác. Tài liệu này ưu tiên *xu hướng & cơ chế* hơn con số tuyệt đối.

---

## Phụ lục B — Bảng mô hình cost-per-document (mẫu điền)

Điền ở **Giai đoạn 0** với khối lượng/tháng thực tế client cung cấp. Số dưới là *placeholder minh họa* — phải thay bằng số đo thật.

| Thành phần | Hóa đơn / AP | KYC onboarding | Ghi chú |
|---|---|---|---|
| Khối lượng/tháng | _<điền>_ | _<điền>_ | Client cung cấp (`scope §3.3`) |
| Ngưỡng chính xác mục tiêu | ~99% (STP 60–80% "sạch") | ~99,5% | Theo rủi ro ([§5.2](#52-ngưỡng-chính-xác-theo-use-case-chốt-ở-giai-đoạn-0)) |
| Cost OCR/trang | _<điền>_ | _<điền>_ | Buy/API (`scope §6`) |
| %VLM × cost VLM/vùng | _<điền>_ | _<điền>_ | Đòn bẩy lớn nhất phần máy |
| Cost LLM trích field (sau cache) | _<điền>_ | _<điền>_ | Trừ prompt cache hit |
| Tỷ lệ vào HITL × cost lao động | **100% GĐ1** × _<đơn giá>_ | **100% GĐ1** × _<đơn giá>_ | Phần lớn cost/doc ban đầu |
| Cost hạ tầng phân bổ | _<điền>_ | _<điền>_ | Queue/storage/gateway |
| **Cost-per-document (GĐ1)** | _<tổng>_ | _<tổng>_ | Phần lớn là lao động HITL |
| **Cost-per-document (GĐ3, nới gating)** | _<tổng giảm>_ | _<giảm ít hơn>_ | %HITL giảm; KYC giảm chậm hơn do ngưỡng cao |
| Trần chi phí/tài liệu (cưỡng chế) | _<biên trên>_ | _<biên trên>_ | Cấu hình tại Gateway ([§8](#8-cost-guardrails-ba-lớp--điểm-cưỡng-chế)) |

> Hai kịch bản bắt buộc tính: **(GĐ1)** 100% HITL — cost cao, ROI từ cycle time + golden data; **(GĐ3)** nới confidence-gating — %HITL giảm kéo cost/doc xuống. So sánh hai cột này cho client thấy *đường đi của ROI theo thời gian*, không phải tiết kiệm tức thì.

---

## Tài liệu liên quan (nội bộ)

- `workflow.md` — Khung kiến trúc (Gateway §8.5, cache §6.6/§6.8, dispatch §4.4, hai lane §6.5, %VLM §10, ADR-1/10/11, roadmap §13).
- `governance.md` — Vòng đời dữ liệu + event schema mang cost telemetry (§4.1 + Phụ lục A: `cost_tokens`, `latency_ms`), observability §6, KPI §9.
- `scope-implementation.md` — Phạm vi & KPI nghiệm thu (cost/page §5, FinOps chargeback out-of-scope §3.2, cảnh báo agentic §6, roadmap §7).
- Tài liệu này — Ràng buộc tài chính & FinOps: dịch kỳ vọng cost-control thị trường → ràng buộc + cơ chế chi phí ở altitude dự án, gắn với đường đánh đổi chính xác–chi phí.

---
author: lytinh358@gmail.com
date: 2026-06-05
status: draft
agents: idp
summary: Business context cho MVP1 Demo (Walking Skeleton / v0.1) — hệ IDP trích xuất hóa đơn end-to-end chạy local; định nghĩa goal, success criteria, scope, requirements, input/output format
---

# MVP1 Demo — Business Context

> **MVP1 Demo = `v0.1 Walking Skeleton`** trong [`../research/roadmap.md`](../research/roadmap.md), ánh xạ GĐ1 (lát mỏng nhất) của [`../research/Architecture_design.md` §13](../research/Architecture_design.md).
> **Một câu:** Chứng minh **giá trị end-to-end** của hệ IDP trên **một luồng happy-path duy nhất** — nhận 1 hóa đơn → OCR → LLM trích field → người duyệt xác nhận → xuất JSON — chạy **hoàn toàn local bằng công cụ free/open-source**, KHÔNG phải production.
> **Tài liệu nền:** scope giao hàng → [`../research/scope-implementation.md`](../research/scope-implementation.md); kiến trúc → [`architecture.md`](architecture.md); dữ liệu/quan sát → [`governance.md`](governance.md).

---

## 1. Goal (mục tiêu)

### 1.1. Mục tiêu chính

Dựng một **lát cắt dọc mỏng nhất chạy được toàn trình** (vertical slice / walking skeleton) cho use case **hóa đơn / Accounts Payable (AP)**, để:

1. **Chứng minh giá trị nghiệp vụ trước stakeholder**: một tài liệu thật chảy qua toàn hệ và ra dữ liệu có cấu trúc dùng được — không phải slide, mà là phần mềm chạy.
2. **Kiểm chứng giả định kỹ thuật cốt lõi**: OCR + LLM trích đúng ~5–8 field quan trọng của hóa đơn ở độ chính xác đủ để bàn luận đi tiếp.
3. **Thu phản hồi sớm** để chốt scope và KPI cho `v0.5 (MVP/Pilot)`: field nào quan trọng, UI duyệt có dùng được không, độ chính xác thô trên mẫu thật ra sao.
4. **Cố định khung kiến trúc** (event/pipeline, contract dữ liệu, contract HITL) ở dạng tối giản để các version sau "dày" lên mà không phải đập đi xây lại.

### 1.2. Phi mục tiêu (Non-goals) — cố ý hoãn

MVP1 **KHÔNG** nhằm:

- Đạt độ chính xác/độ phủ production hay xử lý tài liệu khó (scan mờ, viết tay, layout đa cột, bảng lồng nhau) → để `v2.0` (VLM + region segmentation).
- Có guardrail bảo mật/tuân thủ thật (PII firewall thật, tier enforcement, audit WORM, crypto-shredding) → để `v0.5`/`v1.0`.
- Tích hợp đẩy dữ liệu vào ERP/CRM thật → MVP1 chỉ xuất **file JSON**; integration API đầy đủ để `v0.5`/`v1.0` (ADR-14).
- Xử lý volume / nhiều người dùng đồng thời / autoscale → MVP1 chạy tuần tự, 1–3 tester nội bộ.
- Tự động release (auto-release) bất kỳ output nào → mọi output đều phải qua người duyệt (full-HITL, giữ ADR-11 ngay từ demo).

> **Lằn cắt demo (demo cut-line):** Những gì được phép stub ở MVP1 và mốc bắt buộc bật thật — xem [`../research/roadmap.md` §"Lằn cắt demo"](../research/roadmap.md). **Quy tắc vàng: khoảnh khắc dữ liệu thật/PII xuất hiện, mọi guardrail tương ứng phải bật — không có ngoại lệ "tạm cho demo".**

---

## 2. Success Criteria (tiêu chí thành công)

MVP1 thành công khi **toàn bộ** tiêu chí "phải đạt" dưới đây được thỏa.

### 2.1. Tiêu chí PHẢI đạt (exit criteria — go/no-go sang `v0.5`)

| # | Tiêu chí | Cách đo / Definition of Done |
|---|---|---|
| S1 | **End-to-end chạy được trước stakeholder** trên ≥1 hóa đơn mẫu thật (điều kiện G0-1) | Demo trực tiếp: upload → thấy field trích ra → sửa → xác nhận → tải JSON, không lỗi chặn |
| S2 | **Hai đường input hoạt động**: API (`POST` nộp tài liệu) **và** UI (form upload) | Gọi được cả 2 đường, cùng tạo ra một job xử lý như nhau |
| S3 | **Trích đủ 5–8 field cốt lõi** của hóa đơn với grounding (vùng nguồn) | Mỗi field có `value` + `confidence` + `bbox` để đối chiếu trên ảnh |
| S4 | **UI duyệt tối giản dùng được**: xem field cạnh ảnh gốc, sửa, xác nhận | Reviewer hoàn tất 1 ca duyệt mà không cần hướng dẫn kỹ thuật |
| S5 | **Xuất JSON đúng schema** sau khi reviewer xác nhận | File JSON khớp [contract §6.2](#62-output-format-json) và validate được |
| S6 | **Chạy hoàn toàn local, free/open-source**, không phụ thuộc dịch vụ cloud trả phí | Cài & chạy theo README trên máy sạch, không cần tài khoản cloud |
| S7 | **Bắt được diff predicted-vs-corrected** mỗi lần reviewer sửa | Mỗi chỉnh sửa sinh 1 event log có `value_predicted` + `value_corrected` (mầm golden data) |

### 2.2. Tiêu chí "nên có" (nice-to-have, không chặn)

- Thời gian xử lý 1 hóa đơn 1 trang text rõ ở mức "chấp nhận để demo" (định hướng < 30s không tính thời gian người duyệt).
- Highlight field confidence thấp trong UI để hướng sự chú ý của reviewer.
- Đo thô độ chính xác field-level trên một bộ vài hóa đơn mẫu (baseline tham khảo, **chưa** phải KPI ký nhận).

### 2.3. KHÔNG dùng làm tiêu chí (tránh kỳ vọng sai)

- **Không** lấy "đạt 99% accuracy", "đạt STP%", "throughput ≥30 doc/giờ", "cost-per-document trong biên" làm tiêu chí MVP1 — đó là KPI nghiệm thu của `v0.5`/`v1.0`, đo trên dữ liệu thật sau khi đã ký KPI ở GĐ0.
- **Cảnh báo truyền đạt:** MVP1 dễ bị nhầm là "gần xong". Phải nói rõ với stakeholder: demo **bỏ qua toàn bộ guardrail**; khoảng cách tới production là **phần lớn công sức** (SAD §13.3).

---

## 3. Scope (phạm vi)

### 3.1. In-scope (MVP1 giao)

**Use case:** Hóa đơn / AP duy nhất. Tài liệu: **hóa đơn PDF có text rõ (digital-born hoặc scan chất lượng tốt)**.

**Năng lực chức năng:**

- **Ingestion tối giản:** nhận 1 file PDF qua API + UI; kiểm tra định dạng & kích thước cơ bản; tách trang.
- **OCR:** trích text + layout + bounding box bằng engine open-source (PaddleOCR/Tesseract).
- **Trích field bằng LLM:** map text → ~5–8 field cốt lõi, kèm confidence + grounding bbox.
- **Validation cơ bản:** chuẩn hóa & kiểm tra định dạng số/ngày/tiền (regex + rule tối thiểu).
- **Full HITL tối giản:** UI xem field cạnh ảnh gốc, sửa, xác nhận; **không auto-release**.
- **Output:** xuất file JSON có cấu trúc sau khi reviewer xác nhận.
- **Event log tối giản:** ghi sự kiện pipeline + diff predicted-vs-corrected vào store local (SQLite/JSONL).

**Ngôn ngữ tài liệu:** Tiếng Việt (có dấu) + tiếng Anh, ở mức hóa đơn text rõ.

**Hạ tầng:** chạy **local một process**, pipeline gọi tuần tự; queue in-memory/DB; lưu file local.

### 3.2. Out-of-scope (hoãn sang version sau — cố ý)

| Hạng mục | Lý do hoãn | Bật thật từ |
|---|---|---|
| PII firewall thật, tier enforcement | Demo chỉ dùng dữ liệu tổng hợp/không nhạy cảm, Tier 1 | `v0.5` (text), `v2.0` (image) |
| 4-zone storage, audit WORM, crypto-shredding | Không lưu PII; xóa sau demo | `v0.5` / `v1.0` |
| Integration API đầy đủ (submit/status/result + webhook + ERP push idempotent) | MVP1 chỉ xuất file | `v0.5` (API), `v1.0` (webhook/ERP) |
| Message bus bền, DLQ, saga, idempotency state machine | Chưa chạy volume thật; in-memory chấp nhận mất khi restart | `v1.0` |
| AI Gateway control plane, cost guardrail | Giới hạn tay số doc demo | `v1.0` (ADR-16) |
| VLM, region segmentation, sub-extractor (bảng/biểu đồ/công thức/chữ ký) | Tài liệu khó để sau | `v2.0` |
| Confidence-gating (nới full-HITL) | Chưa có golden data chứng minh | `v3.0` (ADR-11) |
| Phân loại nhiều loại tài liệu, schema management plane | MVP1 cố định 1 loại (hóa đơn) | `v0.5`+ / `v2.0` |
| Autoscale, multi-tenant, SLA/SLO | MVP1 1–3 tester | `v1.0`+ |

### 3.3. Giả định (assumptions)

- Có **bộ hóa đơn mẫu đại diện** ở dạng PDF text rõ để demo (điều kiện G0-1).
- Demo chỉ dùng **dữ liệu tổng hợp / không nhạy cảm** (cho phép stub PII firewall an toàn).
- Máy chạy demo cài được công cụ open-source (Python, engine OCR, LLM local hoặc free-tier).
- Stakeholder hiểu rõ demo là walking-skeleton, không phải sản phẩm production.

---

## 4. Requirements (yêu cầu)

### 4.1. Functional requirements

| ID | Yêu cầu |
|---|---|
| FR-1 | Người dùng nộp 1 hóa đơn PDF qua **UI upload form** → tạo job, trả `job_id`. |
| FR-2 | Hệ cung cấp **API `POST`** nộp tài liệu (multipart hoặc base64) → trả `job_id` (202). |
| FR-3 | Hệ chạy OCR trên tài liệu, sinh text + bounding box. |
| FR-4 | LLM trích **5–8 field cốt lõi** (xem §6.1) từ text, mỗi field có `value`, `confidence`, `grounding_region`. |
| FR-5 | Validation chuẩn hóa & kiểm định dạng ngày/tiền/số hóa đơn; gắn cờ field không hợp lệ. |
| FR-6 | UI hiển thị **ảnh gốc cạnh danh sách field** (pre-fill); highlight field confidence thấp/không hợp lệ. |
| FR-7 | Reviewer **sửa** giá trị field và **xác nhận** ca duyệt. |
| FR-8 | Sau khi xác nhận, hệ **xuất JSON** theo schema §6.2 và cho phép tải về. |
| FR-9 | Mỗi sửa đổi của reviewer sinh **event log** có `value_predicted` + `value_corrected` + ngữ cảnh. |
| FR-10 | Hệ cung cấp **API tra trạng thái/kết quả** job tối giản (`GET status`/`result`). |

### 4.2. Non-functional requirements (mức demo)

| ID | Yêu cầu | Mức demo |
|---|---|---|
| NFR-1 | **Local-first**: chạy không cần cloud trả phí | Bắt buộc |
| NFR-2 | **Reproducible**: cài & chạy theo README trên máy sạch | Bắt buộc |
| NFR-3 | **Traceable**: mọi event có `correlation_id` (theo dõi 1 tài liệu xuyên pipeline) | Bắt buộc |
| NFR-4 | **No auto-release**: không output nào ra ngoài mà chưa qua người duyệt | Bắt buộc (giữ ADR-11) |
| NFR-5 | **Tài liệu là dữ liệu, không phải chỉ thị**: nội dung hóa đơn không bao giờ được diễn giải thành lệnh cho LLM | Bắt buộc (giữ P9) |
| NFR-6 | Độ trễ xử lý 1 trang text rõ | Định hướng < 30s (nice-to-have) |
| NFR-7 | Cấu hình được engine OCR / model LLM qua config, không sửa code | Nên có |

> **Hai bất biến kiến trúc giữ ngay từ demo** (dù mọi thứ khác được stub): **NFR-4 full-HITL** và **NFR-5 tài-liệu-là-dữ-liệu**. Đây là hai trụ an toàn rẻ tiền nhưng quyết định, không được bỏ kể cả ở walking skeleton.

---

## 5. Personas & luồng người dùng (demo)

| Persona | Mục tiêu trong demo |
|---|---|
| **Tester/Dev nội bộ** | Nộp hóa đơn qua API/UI, quan sát pipeline, kiểm tra JSON output |
| **Reviewer (đóng vai)** | Duyệt field cạnh ảnh gốc, sửa sai, xác nhận release |
| **Stakeholder** | Xem demo end-to-end, đánh giá giá trị, ra quyết định go/no-go `v0.5` |

**Happy-path:** Tester upload hóa đơn → hệ OCR + LLM trích field → Reviewer xem cạnh ảnh, sửa 0–N field, xác nhận → hệ xuất JSON → Stakeholder thấy dữ liệu có cấu trúc đúng.

---

## 6. Input & Output Format

### 6.1. Input format

**Tài liệu:**

| Thuộc tính | Giá trị MVP1 |
|---|---|
| Định dạng | **PDF** (text rõ; digital-born hoặc scan tốt). JPG/PNG tùy chọn nếu OCR đọc tốt |
| Số trang | 1 (đa trang: nice-to-have) |
| Kích thước | ≤ ~5MB |
| Ngôn ngữ | Tiếng Việt (có dấu) + tiếng Anh |
| Loại | Hóa đơn / AP |
| Dữ liệu | **Tổng hợp / không nhạy cảm** (bắt buộc ở demo) |

**API submit (tối giản):**

```http
POST /v1/documents
Content-Type: multipart/form-data        # hoặc application/json (base64)

file=<invoice.pdf>
document_type=invoice                     # cố định ở MVP1
```

```jsonc
// Response 202 Accepted
{
  "job_id": "job_01HXY...",
  "status": "queued",
  "created_at": "2026-06-05T08:12:34Z",
  "links": {
    "status": "/v1/documents/job_01HXY.../status",
    "result": "/v1/documents/job_01HXY.../result"
  }
}
```

**~5–8 field cốt lõi cần trích (hóa đơn):**

`invoice_number`, `invoice_date`, `vendor_name`, `vendor_tax_id`, `subtotal`, `tax_amount`, `total_amount`, `currency`.

### 6.2. Output format (JSON)

Output cuối cùng (sau khi reviewer xác nhận) — tập con tối giản của contract trong SAD, **tiến hóa-tương thích** để `v0.5`+ mở rộng (thêm `tenant_id`, `downstream_status`, line-items…):

```jsonc
{
  "job_id": "job_01HXY...",
  "document_type": "invoice",
  "status": "completed",
  "schema_version": "mvp1-result@0.1.0",
  "correlation_id": "corr_8f3a...",
  "source": { "filename": "invoice_demo_001.pdf", "pages": 1 },
  "extraction": {
    "overall_confidence": 0.93,
    "extraction_path": ["ocr", "llm"],
    "fields": [
      {
        "field_name": "invoice_number",
        "value": "INV-2026-001",
        "confidence": 0.98,
        "grounding_region": { "page": 1, "bbox": [420, 85, 160, 22] },
        "validation": { "passed": true, "rules": ["format_invoice_number"] },
        "reviewer_corrected": false
      },
      {
        "field_name": "total_amount",
        "value": 13750000,
        "currency": "VND",
        "confidence": 0.91,
        "grounding_region": { "page": 1, "bbox": [380, 540, 200, 24] },
        "validation": { "passed": true, "rules": ["positive_number", "currency_vnd"] },
        "reviewer_corrected": true
      }
      // ... các field còn lại
    ]
  },
  "hitl_review": {
    "reviewed_at": "2026-06-05T08:13:58Z",
    "reviewer_id": "demo-reviewer",
    "corrections_count": 1
  }
}
```

> Chi tiết **event/log schema** (mỗi field sửa, dwell time, region click) phục vụ governance & golden data: xem [`governance.md`](governance.md).

---

## 7. Out-of-demo handoff (cái gì xảy ra sau MVP1)

| Sau MVP1 → `v0.5` cần | Vì sao |
|---|---|
| **KPI nghiệm thu ký nhận** (STP, first-pass yield, lỗi/1.000, cycle time) | Pilot không có thước đo = vô nghĩa (G0-2/G0-3) |
| **PII firewall text thật** (Presidio + recognizer VN: CCCD/MST/STK) | Chạm dữ liệu thật ⇒ guardrail thật |
| **4-zone storage + chốt nền tảng cloud** | Rời local sang môi trường thật |
| **Integration API `submit/status/result` contract-first (OpenAPI)** | ROI sống/chết ở integration (ADR-14) |
| **Bắt đầu tích lũy golden data đều** | Đường găng tới ROI ở `v3.0` |

---

## 8. Rủi ro & giảm thiểu (cấp demo)

| Rủi ro | Giảm thiểu |
|---|---|
| Nhầm demo là "gần xong" → cắt guardrail khi lên thật | Truyền đạt lằn cắt demo; guardrail là tiêu chí ra `v1.0`, không phải MVP1 |
| Lỡ đưa dữ liệu thật/PII vào khi PII firewall còn stub | **Cấm tuyệt đối**: chỉ dùng dữ liệu tổng hợp ở MVP1 |
| Kỳ vọng accuracy sai (đem mẫu khó vào demo) | MVP1 giới hạn hóa đơn text rõ; tài liệu khó để `v2.0` |
| Scope creep (thêm field/loại tài liệu giữa demo) | Chốt 1 loại + 5–8 field; thay đổi qua change-request sang `v0.5` |

---

## Tài liệu liên quan

- [`architecture.md`](architecture.md) — proposed solution: design concern, infra, tech stack, kiến trúc, sequence diagram, code structure.
- [`governance.md`](governance.md) — data structure, monitoring & observation cho MVP1.
- [`../research/roadmap.md`](../research/roadmap.md) — `v0.1` walking skeleton & lằn cắt demo (chuẩn nền).
- [`../research/scope-implementation.md`](../research/scope-implementation.md) — scope giao hàng & KPI nghiệm thu.
- [`../research/Architecture_design.md`](../research/Architecture_design.md) — SAD hợp nhất (chuẩn cuối cùng).

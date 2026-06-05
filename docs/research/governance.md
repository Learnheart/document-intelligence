# Quản trị, lưu trữ và cải tiến dữ liệu cho hệ IDP/OCR

> Tài liệu tổng hợp từ trao đổi về kiến trúc data extraction (agent extraction + visual grounding), tập trung vào **quy trình quản lý và lưu trữ thông tin dữ liệu** phục vụ ba mục tiêu: **quản trị (governance)**, **lưu trữ (storage)** và **cải tiến trong tương lai (continuous improvement)**.

-----

## 1. Bối cảnh và phạm vi

Hệ thống nền tảng là một pipeline trích xuất dữ liệu tài liệu hiện đại, kết hợp:

- **OCR** cho text in sạch,
- **VLM (vision-language model)** cho layout phức tạp, bảng, biểu đồ,
- **Visual grounding** để định vị từng trường dữ liệu về đúng vùng trên trang (gắn “what” với “where”),
- **Agentic orchestration**: phân loại → trích xuất → kiểm chứng theo schema, có vòng tự sửa và human-in-the-loop (HITL).

So với OCR pipeline truyền thống (chuỗi tuyến tính một chiều, dựa vào template, lỗi lan truyền), hệ này hiểu ngữ nghĩa, thích nghi với định dạng mới và có khả năng tự cải thiện. Khả năng tự cải thiện đó **phụ thuộc trực tiếp vào việc dữ liệu vận hành được quản lý và lưu trữ tốt** — đó là trọng tâm của tài liệu này.

Phạm vi tài liệu: vòng đời của **dữ liệu vận hành** (chỉnh sửa của người dùng, log tương tác, kết quả suy luận, metadata) từ lúc sinh ra → lưu trữ → quản trị → đưa trở lại vòng học để cải tiến.

-----

## 2. Tổng quan luồng dữ liệu cần quản lý

```
Tài liệu vào
   │
   ▼
[Ingestion + phân loại]  ──► sinh log: phân loại, định tuyến
   │
   ▼
[Trích xuất: OCR + VLM + grounding]  ──► sinh log: giá trị dự đoán, confidence, vùng grounding
   │
   ▼
[Kiểm chứng theo schema]  ──► sinh log: pass/fail, điểm tin cậy từng trường
   │
   ├─(đủ tin cậy)─► Hệ thống hạ nguồn (ERP/CRM/kho dữ liệu)  ──► sinh log: export, dùng lại
   │
   └─(tin cậy thấp)─► Người duyệt (HITL)  ──► sinh log: chỉnh sửa cấp trường, accept/reject, dwell time, vùng click
                          │
                          ▼
                  Kho dữ liệu vận hành (logs + corrections)
                          │
                          ▼
                  Vòng cải tiến (fast-path memory / slow-path retrain)
```

Mọi mũi tên trong sơ đồ đều phát ra sự kiện. Quản trị tốt nghĩa là **bắt, lưu, làm sạch và truy vết được** tất cả các sự kiện này.

-----

## 3. Nguồn dữ liệu và loại tín hiệu

Có hai nhóm tín hiệu, cần lưu trữ với cùng một schema thống nhất để khai thác được:

### 3.1. Tín hiệu tường minh (explicit)

- **Chỉnh sửa cấp trường**: giá trị model dự đoán vs giá trị người sửa (tín hiệu vàng).
- **Accept không sửa**: tín hiệu dương rằng trích xuất đúng.
- **Reject / chạy lại / upload lại**: tín hiệu âm mạnh.
- Đánh giá, gắn cờ, ghi chú của người duyệt.

### 3.2. Tín hiệu ngầm (implicit) — sinh ra từ log hành vi

- **Vùng người dùng click để kiểm tra** (nhờ visual grounding): cho biết model nên chú ý ở đâu.
- **Dwell time** trên một trường trước khi xác nhận (dừng lâu = nghi ngờ).
- **Thứ tự sửa**, **truy vấn tìm kiếm lặp lại**, **hành vi hạ nguồn** (copy, export, chỉnh lại sau khi đẩy đi).
- **Abandonment** (bỏ dở phiên).

> Nguyên tắc: tín hiệu ngầm dồi dào và không gây phiền người dùng, nhưng **nhiễu và mơ hồ** — phần lớn công sức quản trị là làm sạch sự mơ hồ đó (xem mục 6 và 7).

-----

## 4. Quy trình quản lý và lưu trữ dữ liệu (trọng tâm)

### 4.1. Chuẩn hóa schema sự kiện — điều kiện tiên quyết

Log chỉ khai thác được nếu có cấu trúc. Định nghĩa một **event schema** thống nhất, **đánh version cho chính schema** (vì nó sẽ tiến hóa), và ghi đầy đủ ngữ cảnh suy luận để có thể **replay** (dựng lại vì sao model ra kết quả đó).

Mẫu schema sự kiện (tham khảo — xem Phụ lục A để đầy đủ):

```json
{
  "event_id": "evt_...",
  "event_type": "field_edited | field_accepted | field_rejected | region_clicked | document_rejected | exported | rerun",
  "timestamp": "ISO-8601",
  "session_id": "ses_...",
  "document_id": "doc_...",
  "document_type": "invoice | contract | claim | ...",
  "vendor": "...",
  "user_id": "pseudonymized",
  "role": "reviewer | analyst | ...",
  "model_version": "extractor@x.y.z",
  "pipeline_version": "pipe@a.b.c",
  "schema_version": "evt-schema@1.3",
  "field_name": "total_amount",
  "value_predicted": "...",
  "value_corrected": "...",
  "confidence_score": 0.0,
  "grounding_region": { "page": 1, "bbox": [x0, y0, x1, y1] },
  "dwell_time_ms": 0,
  "sensitivity_label": "public | internal | pii | phi | financial",
  "consent_flags": ["..."]
}
```

### 4.2. Phân tầng lưu trữ và vòng đời (lifecycle / retention)

Không lưu mọi thứ ở một chỗ mãi mãi. Phân tầng theo tần suất truy cập và yêu cầu tuân thủ:

|Tầng              |Nội dung                                       |Thời gian (tham khảo)|Hạ tầng điển hình           |Mục đích                           |
|------------------|-----------------------------------------------|---------------------|----------------------------|-----------------------------------|
|**Hot**           |Log gần đây, corrections đang dùng để học/debug|0–90 ngày            |DB nhanh / object store nóng|Học fast-path, debug, observability|
|**Warm**          |Log tích lũy cho retrain định kỳ               |3–12 tháng           |Data lake / lakehouse       |Slow-path retrain, phân tích       |
|**Cold / Archive**|Log cũ, lưu để tuân thủ/audit                  |1–7 năm (tùy ngành)  |Object store lạnh, giá rẻ   |Bằng chứng kiểm toán, truy vết     |
|**Xóa**           |Sau hết hạn retention                          |—                    |—                           |Tuân thủ, kiểm soát chi phí        |

**Yêu cầu cốt lõi**: chính sách xóa phải *chứng minh được* (provable deletion) — đây là nơi lineage hỗ trợ (mục 4.4).

### 4.3. Versioning dữ liệu và feature store

Để vòng học có thể **tái lập (reproducible)** và **rollback**, dữ liệu phải được đánh version như code.

- **Cách tiếp cận**: nhân bản toàn bộ dataset; thêm trường `valid_from`/`valid_to` để “time travel”; hoặc dùng công cụ kiểu Git áp branch/commit lên dữ liệu.
- **Chọn công cụ theo quy mô**: DVC hợp dự án nhỏ/dữ liệu bảng; lakeFS hợp data lake khối lượng lớn (ảnh/log terabyte); Pachyderm cho lineage theo container.
- **Feature store**: biến log thô thành feature tái sử dụng, đảm bảo **nhất quán train/serve** và truy vết **feature → model**. Đây là thứ ngăn lỗi kinh điển “feature lúc huấn luyện khác lúc chạy thật”.

### 4.4. Lineage — xương sống của truy vết và tuân thủ

Lineage trả lời: dữ liệu **đến từ đâu**, **biến đổi ra sao**, **model version nào** đã dùng nó, **ai tiêu thụ** đầu ra.

- Mở rộng lineage tới **pipeline ML, feature store và model registry**, không chỉ hạ tầng dữ liệu truyền thống.
- Coi lineage là **tài sản sống**: version control cho metadata lineage, tự động phát hiện thay đổi, tích hợp CI/CD.
- Giá trị trực tiếp cho IDP ở BFSI/y tế: lineage cung cấp **bằng chứng về cách xử lý, lưu giữ và xóa dữ liệu** khi bị kiểm toán.
- Lý tưởng: lineage và observability **chia sẻ metadata**, để khi cảnh báo nổ thì ngữ cảnh lineage có sẵn ngay.

-----

## 5. Quản trị, phân quyền và quyền riêng tư

Đây là ràng buộc cứng vì log IDP thường chứa dữ liệu nhạy cảm (hồ sơ tài chính, bệnh án). Đặc biệt quan trọng vì BFSI và y tế là hai ngành ứng dụng IDP nhiều nhất / tăng nhanh nhất.

- **Làm sạch/ẩn danh PII *trước* khi log** (sanitization, pseudonymization, tokenization) — giữ giá trị học nhưng loại định danh.
- **Gắn nhãn phân loại độ nhạy cảm** và để chính sách đi theo dữ liệu: tự động mang nhãn nhạy cảm + tag quản trị **dọc theo đường lineage** (tag propagation), đơn giản hóa thực thi xử lý PII và hạn chế truy cập.
- **Kết hợp lineage với metadata nghiệp vụ**: định nghĩa dữ liệu, phân loại, business rule, quyền sở hữu — vừa có khả năng nhìn vừa có ngữ cảnh.
- **Phân quyền truy cập** theo vai trò; nguyên tắc tối thiểu quyền (least privilege).
- **Audit & tuân thủ**: với use case bị quản lý chặt, duy trì **model card, metadata lineage, access log**; tự động thu thập **artifact bằng chứng** cho kiểm toán (dataset huấn luyện, kết quả validation, phê duyệt, mốc triển khai).
- **Quyền của chủ thể dữ liệu**: hỗ trợ right-to-deletion, consent management theo quy định áp dụng (ví dụ GDPR).

-----

## 6. Observability và chất lượng dữ liệu

Log là dữ liệu sống nên phải giám sát sức khỏe của nó.

- **So sánh phân phối train vs serve**: xuất feature dùng khi suy luận sang nền tảng observability để phát hiện lệch.
- **Phát hiện drift** (phân phối tài liệu đổi → độ chính xác tụt) và **anomaly** (ví dụ tỷ lệ chỉnh sửa của một vendor tăng đột biến).
- **Ngưỡng + runbook cho cảnh báo**: định nghĩa rõ khi nào kích hoạt điều tra hoặc retrain.
- **Nền tảng tham khảo**: Arize, WhyLabs, Fiddler để tập trung cảnh báo và phân tích nguyên nhân gốc.
- **Chất lượng dữ liệu vào vòng học**: kiểm tra hoàn chỉnh, hợp lệ, trùng lặp, mâu thuẫn trước khi nạp.

-----

## 7. Cơ chế cải tiến trong tương lai (continuous improvement)

Mục tiêu: hệ tốt lên theo thời gian, lý tưởng là **chỉ nhờ được dùng**, mà không tăng gánh nặng cho người dùng.

### 7.1. Biến chỉnh sửa thành tín hiệu học (HITL → active learning)

- Bắt **diff** (predicted vs corrected) kèm ngữ cảnh, không chỉ kết quả cuối.
- **Active learning / lấy mẫu theo độ bất định**: dồn công sức người duyệt vào các ca bất định/tác động cao và edge case, thay vì retrain mù quáng.

### 7.2. Hai “tốc độ” cập nhật

- **Fast-path (tức thì)**: cơ chế “bộ nhớ” kiểu RAG/few-shot hấp thụ chỉnh sửa ngay — sửa layout của một vendor hôm nay, áp dụng cho tài liệu tương tự ngày mai, không cần retrain.
- **Slow-path (định kỳ)**: gom corrections, fine-tune/retrain để củng cố.
- Phối hợp cả hai là kiến trúc lý tưởng: thích nghi ngay + củng cố bền vững.

### 7.3. Học từ log hành vi (implicit feedback)

- Khai thác tín hiệu ngầm trong log: chú ý **nội dung** của phản hồi, không chỉ cực tính (dương/âm).
- **Interaction-Grounded Learning (IGL)**: học hàm thưởng **cá nhân hóa cho từng người dùng** thay vì giả định cố định (cùng hành vi có thể mang ý nghĩa khác nhau giữa các người dùng).
- **Online learning**: tinh chỉnh hồ sơ per-user/per-tổ chức theo thời gian thực (cấu hình trích xuất, ngưỡng tin cậy riêng).
- Lưu ý giới hạn của log tĩnh: không tự thích nghi với người dùng mới (chưa có lịch sử) và không học được từ sửa lỗi thời gian thực → cần kết hợp fast-path.

### 7.4. Xử lý nhiễu và chống quên

- **Lọc nhiễu**: không tin mọi chỉnh sửa là đúng; đối chiếu nhiều người duyệt hoặc tính nhất quán theo thời gian trước khi học.
- **Chống catastrophic forgetting**: cập nhật mới không xóa kiến thức cũ (continual learning có cơ chế giữ kiến thức trước).
- **Đánh giá trên tập holdout** trước khi đẩy lên production — tránh “tự tin tăng nhưng độ chính xác giảm”.

-----

## 8. Lộ trình triển khai theo mức độ trưởng thành

Đừng over-engineer. Mỗi lớp đều thêm độ phức tạp hạ tầng cần thiết lập và bảo trì. Triển khai theo giai đoạn:

**Giai đoạn 1 — Tối thiểu khả thi (bắt buộc)**

- Event schema chuẩn hóa + version.
- PII sanitization trước khi log.
- Retention/tiering policy cơ bản.

**Giai đoạn 2 — Truy vết & tái lập**

- Data versioning + feature store.
- Lineage cơ bản (dữ liệu → feature → model).
- Audit trail + access control.

**Giai đoạn 3 — Tự cải thiện & quan sát**

- Observability + drift detection + runbook.
- Active learning sampling.
- Fast-path memory (RAG) + slow-path retrain định kỳ.

**Giai đoạn 4 — Cá nhân hóa nâng cao**

- Học từ implicit feedback / IGL.
- Online per-user/per-tổ chức.
- Lọc nhiễu nâng cao, continual learning chống quên.

-----

## 9. Chỉ số đo lường (KPIs)

|Nhóm                |Chỉ số                                   |Ý nghĩa                                                 |
|--------------------|-----------------------------------------|--------------------------------------------------------|
|**UX / vận hành**   |Intervention rate (tỷ lệ can thiệp tay)  |Mục tiêu giảm dần theo thời gian                        |
|                    |Time-to-correct                          |Thời gian người duyệt xử lý một ca                      |
|                    |Repeat-error rate                        |Cùng một lỗi có bị lặp lại không (đo hiệu quả fast-path)|
|**Chất lượng model**|Field-level accuracy / precision / recall|Theo loại tài liệu, vendor                              |
|                    |Confidence calibration                   |Độ tin cậy có khớp độ chính xác thực                    |
|**Dữ liệu**         |Data drift score                         |Lệch phân phối train vs serve                           |
|                    |Data quality (completeness, validity)    |Sức khỏe dữ liệu vào vòng học                           |
|**Quản trị**        |% log đã gắn nhãn nhạy cảm               |Mức độ phủ governance                                   |
|                    |Tỷ lệ artifact audit tự động hóa         |Sẵn sàng kiểm toán                                      |

-----

## 10. Cạm bẫy và rủi ro cần canh

- **Nhiễu/mơ hồ của tín hiệu ngầm**: dễ đầu độc mô hình nếu không lọc.
- **Bias** (thiên lệch về vendor/định dạng phổ biến): model giỏi phần phổ biến nhưng yếu phần đuôi.
- **Attribution sai**: “accept” có thể vì đúng, cũng có thể vì người dùng không buồn kiểm tra.
- **Quyền riêng tư**: log là dữ liệu người dùng — vi phạm tuân thủ là rủi ro pháp lý nghiêm trọng (đặc biệt BFSI/y tế).
- **Over-engineering**: dựng quá nhiều lớp hạ tầng trước khi cần.
- **Lineage/observability tách rời**: khi cảnh báo nổ mà không có ngữ cảnh lineage thì khó truy nguyên.

-----

## Phụ lục A — Mẫu event schema đầy đủ (gợi ý mở rộng)

Ngoài các trường ở mục 4.1, cân nhắc bổ sung:

- `tenant_id` / `org_id`: tách dữ liệu theo khách hàng (multi-tenant).
- `latency_ms`, `cost_tokens`: phục vụ tối ưu chi phí/hiệu năng.
- `validation_result`: chi tiết pass/fail theo từng rule schema.
- `correction_source`: người duyệt sửa hay rule tự sửa.
- `downstream_status`: trạng thái ở hệ hạ nguồn (đẩy thành công, bị từ chối…).
- `data_retention_class`: liên kết trực tiếp tới chính sách tiering ở mục 4.2.

## Phụ lục B — Bộ công cụ tham khảo (2026)

|Lớp                           |Công cụ ví dụ                                                |
|------------------------------|-------------------------------------------------------------|
|Experiment tracking / registry|MLflow (+ Unity Catalog)                                     |
|Data versioning               |DVC, lakeFS, Dolt                                            |
|Lineage                       |Unity Catalog, Pachyderm, Atlan                              |
|Feature store                 |Feast, Tecton                                                |
|Observability ML              |Arize, WhyLabs, Fiddler, Monte Carlo                         |
|Nền tảng cloud IDP            |AWS Textract, Google Document AI, Azure Document Intelligence|


> Lưu ý: việc chọn nền tảng cloud trong thực tế bị chi phối bởi hệ sinh thái sẵn có nhiều hơn là chênh lệch độ chính xác.

-----

## Tài liệu tham khảo

**Học liên tục & human-in-the-loop**

- Mindee — Human-in-the-loop & continuous learning: <https://www.mindee.com/blog/what-is-human-in-the-loop-automation>
- Devoteam — HITL, active learning: <https://www.devoteam.com/expert-view/human-in-the-loop-what-how-and-why/>
- Ultralytics — HITL machine learning: <https://www.ultralytics.com/blog/human-in-the-loop-machine-learning>
- Encord — HITL explained: <https://encord.com/blog/human-in-the-loop-ai/>
- ABBYY — OCR vs IDP: <https://www.abbyy.com/blog/ocr-vs-idp/>
- RiCL — Reinforced interactive continual learning (noisy feedback): <https://arxiv.org/pdf/2505.09925>

**Học từ log tương tác / implicit feedback**

- Implicit user feedback from interaction logs (NYU): <https://arxiv.org/pdf/2507.23158>
- Interaction-Grounded Learning, personalized reward (Microsoft Research): <https://arxiv.org/pdf/2211.15823>
- Personalized Agents from Human Feedback (PAHF): <https://arxiv.org/html/2602.16173v1>
- Dynamic personalization via continuous feedback loops: <https://arxiv.org/html/2602.23376>
- Implicit feedback in recommender systems (Jannach et al.): <https://web-ainf.aau.at/pub/jannach/files/BookChapter_Social_Information_Access_2018.pdf>
- HumAIne — real-time personalization via online RL: <https://arxiv.org/html/2509.04303v1>
- SymbioticRAG — interaction logging cho personalized retrieval: <https://arxiv.org/pdf/2505.02418>

**Quản trị, lưu trữ & MLOps**

- Data versioning best practices (Label Your Data): <https://labelyourdata.com/articles/machine-learning/data-versioning>
- Databricks data lineage best practices (Atlan): <https://atlan.com/know/databricks-data-lineage-best-practices/>
- Data lineage best practices (OvalEdge): <https://www.ovaledge.com/blog/data-lineage-best-practices>
- Data lineage tools 2026 (Atlan): <https://atlan.com/data-lineage-tools/>
- Data lineage concepts (Decube): <https://www.decube.io/post/data-lineage-concepts>
- Top MLOps tools 2026 (Medium / Online Inference): <https://medium.com/online-inference/top-mlops-tools-in-2026-858fd479acac>
- MLOps in 2026 — definitive guide: <https://rahulkolekar.com/mlops-in-2026-the-definitive-guide-tools-cloud-platforms-architectures-and-a-practical-playbook/>

**Kiến trúc trích xuất (bối cảnh)**

- Operationalizing Document AI — microservice architecture: <https://arxiv.org/html/2605.18818v1>
- NVIDIA RAG blueprint: <https://github.com/NVIDIA-AI-Blueprints/rag>
- LlamaIndex — Agentic Document Extraction: <https://www.llamaindex.ai/blog/agentic-document-extraction>

*Lưu ý: các nguồn từ blog nhà cung cấp mang tính giới thiệu sản phẩm; nên đối chiếu với paper học thuật khi cần độ chính xác kỹ thuật và số liệu benchmark.*

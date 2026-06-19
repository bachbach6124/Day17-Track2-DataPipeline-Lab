# Bonus Design — Pipeline tri thức văn bản pháp luật Việt Nam

## 1. Bài toán và ràng buộc thực tế

Tôi muốn xây một trợ lý tra cứu quy định nội bộ cho bộ phận pháp chế và vận hành
của một doanh nghiệp tại Việt Nam. Người dùng cần hỏi các câu như:

- “Hợp đồng loại này cần được lưu bao lâu?”
- “Quy định mới thay thế văn bản nào?”
- “Nghĩa vụ của chi nhánh phụ thuộc vào những điều khoản nào?”
- “Tại ngày ký hợp đồng, quy định nào đang có hiệu lực?”

Dữ liệu đầu vào gồm PDF văn bản pháp luật, công văn, quy trình nội bộ và phụ lục
hợp đồng. Nhiều PDF là bản scan, có bảng, hai cột, header lặp lại, số điều khoản,
chữ ký và dấu. Một văn bản có thể được sửa đổi hoặc thay thế bởi văn bản khác.
Tên cơ quan, số hiệu và cách viết ngày tháng không hoàn toàn thống nhất. Tài liệu
nội bộ còn có thể chứa thông tin cá nhân hoặc bí mật kinh doanh.

Người dùng không chỉ cần câu trả lời nhanh mà còn cần nguồn chứng minh: số văn
bản, điều khoản, phiên bản và trang gốc. Một câu trả lời đúng nội dung nhưng dùng
văn bản chưa có hiệu lực tại thời điểm được hỏi vẫn là câu trả lời sai. Vì vậy,
đây không chỉ là bài toán tìm kiếm semantic; pipeline phải giữ provenance,
versioning và point-in-time correctness.

Mục tiêu ban đầu là khoảng 100.000 tài liệu, cập nhật hằng ngày, phục vụ 200
người dùng nội bộ. Hệ thống ưu tiên độ tin cậy và khả năng audit hơn độ trễ tuyệt
đối. P95 dưới 5 giây là chấp nhận được nếu câu trả lời có trích dẫn rõ ràng.

## 2. Nguồn và hình dạng dữ liệu: giữ bản gốc hay chuẩn hóa ngay?

**Quyết định: giữ Bronze bất biến, sau đó mới chuẩn hóa thành Silver.**

Trade-off là tốn thêm dung lượng lưu trữ và phải quản lý nhiều phiên bản, đổi lại
có thể tái xử lý khi OCR hoặc parser được cải thiện. Bronze lưu file gốc, checksum,
URL nguồn, thời điểm tải, MIME type và metadata HTTP. Không sửa trực tiếp PDF đã
ingest.

Silver chứa text đã OCR, block layout, bảng, số trang, tiêu đề và các trường đã
chuẩn hóa như `document_number`, `issued_at`, `effective_from`,
`effective_to`, `issuer` và `supersedes`.

Mỗi tài liệu được định danh bằng content hash. Nếu crawler tải lại cùng một file,
pipeline bỏ qua thay vì tạo bản sao. Nếu URL cũ trả về nội dung mới, hệ thống tạo
version mới và giữ version cũ. Thiết kế này đắt hơn overwrite nhưng cần thiết để
audit câu trả lời lịch sử.

## 3. Batch hay streaming?

**Quyết định: micro-batch hằng ngày, không dùng streaming toàn phần.**

Streaming giảm độ trễ cập nhật nhưng tăng đáng kể độ phức tạp của OCR, retry,
ordering và version reconciliation. Nguồn pháp luật không phát sinh hàng nghìn
sự kiện mỗi giây; yêu cầu thực tế chỉ là tài liệu mới có thể tìm kiếm trong ngày.
Do đó, crawler chạy mỗi 6 giờ và pipeline xử lý theo micro-batch. Tài liệu được
đánh dấu khẩn có thể kích hoạt một run riêng.

Batch cũng giúp gom tài liệu để tối ưu sử dụng GPU OCR/embedding. Nhược điểm là
tài liệu mới có thể chậm vài giờ. Tôi chấp nhận độ trễ đó và hiển thị
`last_successful_ingestion_at` trong giao diện để người dùng biết độ mới của kho.

Phương án **streaming từng trang ngay khi tải được** bị loại vì không tạo thêm giá
trị tương xứng: một tài liệu chưa xử lý đủ trang có thể xuất hiện như phiên bản
hoàn chỉnh, gây câu trả lời thiếu điều khoản hoặc thiếu phụ lục.

## 4. Hợp đồng dữ liệu và quality gate

**Quyết định: quarantine theo tài liệu, nhưng cho phép xử lý một phần có gắn cờ.**

Các kiểm tra bắt buộc trước khi tài liệu vào serving index gồm:

- Có checksum, nguồn và thời điểm ingest.
- OCR coverage đạt ngưỡng tối thiểu; text không chỉ gồm ký tự rác.
- Số trang parser đọc được khớp PDF.
- Điều khoản và số hiệu không bị trùng bất thường.
- Ngày hiệu lực không trước ngày ban hành nếu không có lý do được ghi nhận.
- Mỗi chunk giữ được `document_id`, version, trang và vị trí block.
- Dữ liệu nhạy cảm được phân loại trước khi đưa sang dịch vụ bên ngoài.

Tài liệu hỏng hoàn toàn đi vào quarantine và không được index. Tài liệu chỉ hỏng
một bảng hoặc một trang vẫn được lưu ở Silver với `quality_status=partial`, nhưng
retriever mặc định hạ điểm và giao diện cảnh báo người dùng.

Quarantine tăng hơn 5% trong một batch sẽ gửi cảnh báo cho data owner. Tôi chọn
quarantine thay vì dừng toàn pipeline vì một PDF lỗi không nên làm chậm hàng
nghìn tài liệu tốt. Tuy nhiên, lỗi contract ở cấp hệ thống, chẳng hạn mọi tài
liệu đều mất provenance, phải làm cả batch thất bại.

## 5. Vector RAG hay knowledge graph?

**Quyết định: dùng hybrid retrieval, nhưng chỉ tạo graph cho quan hệ có giá trị.**

Vector search phù hợp với câu hỏi lookup như “Điều khoản nói gì về thời hạn
thanh toán?”. Nó chịu được cách diễn đạt khác nhau và triển khai rẻ hơn graph.
Knowledge graph phù hợp với câu hỏi multi-hop như “Văn bản A sửa điều nào của B,
B đang áp dụng cho đơn vị nào, và tại ngày X quy định nào còn hiệu lực?”.

Graph lưu các quan hệ có schema giới hạn:

```text
(Document)-[:AMENDS]->(Document)
(Document)-[:REPLACES]->(Document)
(Clause)-[:BELONGS_TO]->(Document)
(Clause)-[:REFERS_TO]->(Clause)
(Document)-[:ISSUED_BY]->(Organization)
```

Tôi không đưa mọi danh từ thành entity. Graph “trích tất cả” bị loại vì entity
resolution tiếng Việt dễ tạo hàng nghìn node trùng, chi phí review lớn và truy
vấn trở nên nhiễu. Chỉ các quan hệ phục vụ câu hỏi nghiệp vụ đã biết mới được
extract. Kết quả graph dùng để mở rộng tập tài liệu ứng viên; vector retriever
sau đó chọn chunk, và câu trả lời cuối luôn trích dẫn chunk gốc.

## 6. Point-in-time và chống leakage

**Quyết định: mọi retrieval pháp lý phải nhận `as_of_date`.**

Mỗi version có `effective_from`, `effective_to` và `observed_at`. Khi người dùng
hỏi về một hợp đồng ký ngày 10/01, retriever chỉ được thấy văn bản có hiệu lực tại
ngày đó. Không thể chỉ lấy version mới nhất.

Điều này giống ASOF JOIN trong lab: câu hỏi là event, còn hiệu lực văn bản là
feature history. Nếu join ngây thơ, hệ thống có thể dùng quy định ban hành tháng
6 để đánh giá một quyết định từ tháng 1. Offline evaluation cũng phải đóng băng
corpus theo thời điểm tạo câu hỏi; nếu eval cũ vô tình nhìn thấy văn bản tương
lai, metric sẽ nói dối.

## 7. Flywheel, failure semantics và chi phí

**Quyết định: feedback sản xuất tạo candidate data, không tự động thành ground
truth.**

Pipeline ghi trace gồm câu hỏi, tài liệu được retrieve, graph path, câu trả lời,
trích dẫn, latency và feedback. Những lượt được chuyên viên pháp chế sửa sẽ trở
thành candidate cho eval hoặc preference pair. Một người duyệt phải xác nhận
`chosen`; thumbs-up đơn lẻ không đủ vì người dùng có thể hài lòng với câu trả lời
nghe hợp lý nhưng sai nguồn.

Eval set được version hóa và tách theo loại câu hỏi, thời gian và cơ quan ban
hành. Trước khi tạo dữ liệu train, pipeline decontaminate bằng exact match,
normalized Vietnamese text và similarity để loại cả prompt paraphrase gần với
eval. Nếu bỏ bước này, model có thể nhớ benchmark và metric tăng dù khả năng xử
lý văn bản mới không đổi.

### Vận hành, backfill và ngân sách

Mỗi stage ghi output vào version tạm, kiểm tra row count và checksum rồi mới
atomic publish. Retry dùng cùng `run_id` và content hash nên idempotent. Index
vector hoặc graph chỉ chuyển alias sang version mới sau khi build hoàn chỉnh;
không để người dùng truy vấn index nửa cũ nửa mới.

Backfill theo `document_id` hoặc khoảng ngày, không xóa dữ liệu đang serving.
Nếu extractor mới tạo kết quả kém hơn, alias được rollback về version trước.

Chi phí lớn nhất dự kiến là OCR và embedding, không phải lưu Bronze. Tôi giảm chi
phí bằng content hash, chỉ OCR trang thay đổi, batch GPU và cache embedding theo
`model_version + chunk_hash`. Tài liệu nhạy cảm ưu tiên OCR/embedding trong hạ
tầng kiểm soát được; không gửi ra API ngoài chỉ vì rẻ hơn. Đây là đánh đổi giữa
chi phí vận hành và yêu cầu bảo vệ dữ liệu trong bối cảnh Việt Nam.

## 8. Sơ đồ kiến trúc

```text
Web/PDF/S3 sources
        |
        v
Crawler + checksum ---------> Bronze immutable object store
        |                              |
        v                              | replay/backfill
OCR + layout parser <------------------+
        |
        v
Schema/quality/PII gate ----bad------> Quarantine + alert
        |
       good/partial
        v
Silver: versioned documents + clauses + provenance
        |
        +--------------+-------------------+
        |              |                   |
        v              v                   v
Vector chunks     Relation extractor   Effective-date history
        |              |                   |
        v              v                   |
Vector index      Knowledge graph          |
        +--------------+-------------------+
                       |
                       v
        Hybrid retriever + ASOF filter
                       |
                       v
          Answer + citations + trace
                       |
                       v
      Human review -> eval/DPO candidates
                       |
                       v
          Decontamination -> datasets
```

Thiết kế này ưu tiên ba invariant: không mất provenance, không dùng tri thức từ
tương lai và không biến feedback chưa kiểm chứng thành dữ liệu huấn luyện. Chúng
làm pipeline phức tạp hơn một vector database đơn giản, nhưng trực tiếp bảo vệ độ
tin cậy của sản phẩm.

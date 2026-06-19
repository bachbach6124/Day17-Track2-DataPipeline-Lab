# Reflection — Day 17

1. Bước dễ hỏng âm thầm nhất là curate Bronze thành eval/DPO: pipeline vẫn chạy
nhưng có thể ghép sai prompt, chosen và rejected. Tôi sẽ kiểm tra schema, tỷ lệ
outcome, số pair, lấy mẫu thủ công và đặt invariant: chosen từ lượt thành công,
rejected từ lượt lỗi cùng prompt.

2. Nếu bỏ decontamination, model được train trên câu hỏi dùng để chấm và có thể
ghi nhớ đáp án thay vì tổng quát hóa. Điểm eval tăng giả, còn điểm trên holdout
mới hoặc prompt paraphrase không tăng tương ứng.

3. Trong hệ thống chống gian lận, `chargeback_count` rất nguy hiểm. Giao dịch
ngày 1 không được nhìn chargeback phát sinh ngày 20; thiếu ASOF khiến model học
từ tương lai và kết quả offline tốt hơn serving thật.

4. Graph trả lời tốt “Widget được giao từ đâu?” qua
`widget → accessory → Hanoi`; vector khó nối hai fact ở hai chunk. Với câu
“Widget được hoàn trả trong bao lâu?”, một chunk đã đủ nên graph là thừa.

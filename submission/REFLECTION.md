# Reflection — Day 17 (≤ 200 words)

Answer briefly, in your own words. This is graded on reasoning, not length.

1. **The flywheel.** Day 13 emitted agent traces; today you turned them into an
   eval set and DPO pairs that Day 22 will train on. Which step in
   `traces → Bronze → datasets` would break most silently in production if you
   got it wrong — and how would you detect it?

2. **Decontamination.** Your run dropped 2 of 3 preference pairs because their
   prompts were in the eval set. What concretely goes wrong if you *skip* this
   step and train on those pairs? How would the lie show up in your metrics?

3. **Point-in-time.** The naive join leaked a future `lifetime_spend` into the
   training row. Describe one feature in a system you know that would be
   dangerous to join without an `ASOF`/point-in-time guard.

4. **Graph vs vector.** From `kg_demo.py`, name one question the knowledge graph
   answers well that flat chunk retrieval (`embed.py`) would struggle with, and
   one where the graph is overkill.

_Write your answers below._

Trần Văn Khoa - 2A202600827

1. Bước dễ hỏng âm thầm nhất là flatten trace tree sang Bronze. Nếu thiếu span con hoặc sai parent/trace_id, dashboard vẫn có dòng nhưng cost, latency và outcome bị lệch. Tôi sẽ phát hiện bằng check số span/trace, tỉ lệ trace thiếu root, và so sánh rollup với log gốc.

2. Nếu bỏ decontamination, model sẽ học lại prompt nằm trong eval set. Điểm eval tăng giả vì model đã thấy câu hỏi/đáp án trong train, nhưng khi gặp prompt mới ngoài phân phối thì chất lượng không tăng tương ứng.

3. Với hệ thống chấm điểm tín dụng hoặc chống gian lận, feature như tổng chi tiêu, số lần hoàn tiền, hoặc số giao dịch bị dispute phải join theo thời điểm quyết định. Join naive có thể lấy thông tin xảy ra sau đơn vay/giao dịch.

4. KG trả lời tốt câu multi-hop như widget liên quan accessory nào và nằm ở warehouse nào. Vector retrieval phù hợp hơn cho câu lookup trực tiếp như chính sách đổi trả widget trong bao nhiêu ngày; dùng graph cho câu đó là quá nặng.

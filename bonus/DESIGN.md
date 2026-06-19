# Bonus Design - Flywheel dữ liệu cho chatbot CSKH thương mại điện tử Việt Nam

Sinh viên: Trần Văn Khoa - 2A202600827

## Bài toán và ràng buộc

Tôi chọn bài toán xây pipeline dữ liệu cho một chatbot chăm sóc khách hàng của sàn thương mại điện tử tại Việt Nam. Chatbot trả lời các câu hỏi về đổi trả, bảo hành, trạng thái đơn hàng, voucher, hoàn tiền và khiếu nại. Dữ liệu đầu vào gồm hội thoại người dùng, trace của agent, kết quả gọi tool, tài liệu chính sách nội bộ, lịch sử đơn hàng và feedback sau cuộc trò chuyện. Khó khăn nằm ở chỗ dữ liệu tiếng Việt có nhiều cách viết không dấu, viết tắt, lẫn tiếng Anh, lỗi chính tả, và nhiều thông tin nhạy cảm như số điện thoại, địa chỉ, mã đơn hàng. Pipeline không chỉ cần phục vụ RAG hiện tại mà còn phải biến traffic thật thành eval set và dữ liệu preference để cải thiện model sau này.

Mục tiêu không phải là gom thật nhiều log bằng mọi giá. Mục tiêu là tạo một vòng lặp an toàn: production trace đi vào Bronze, được validate và ẩn PII, sau đó tách thành eval set, preference pairs, feature theo thời điểm, và dashboard chất lượng. Nếu làm sai, điểm eval có thể tăng giả, model có thể học dữ liệu riêng tư, hoặc training dùng thông tin tương lai làm kết quả offline đẹp nhưng production kém.

## Kiến trúc đề xuất

```text
chatbot traffic
   |
   v
raw traces + feedback + tool calls
   |
   v
Bronze trace store  ---> PII redaction / schema validation ---> quarantine
   |
   +--> span flatten + per-trace summary ---> quality dashboard
   |
   +--> held-out eval curation --------+
   |                                   |
   +--> DPO pair mining --> decontam --+--> training/eval datasets
   |
   +--> PIT feature builder --> model analytics

policy docs ---> chunk/embed ---> vector index
policy docs ---> triple extraction ---> knowledge graph
```

## 1. Nguồn dữ liệu và schema drift

Dữ liệu chính đến từ trace của agent: user message, model output, tool name, tool result, latency, token, outcome, user feedback và escalation flag. Tôi sẽ lưu raw event vào Bronze gần nguyên bản, nhưng bắt buộc có contract tối thiểu: `trace_id`, `span_id`, `timestamp`, `role`, `content_type`, `outcome`. Tradeoff là strict schema giúp phát hiện lỗi sớm, nhưng quá strict sẽ làm rớt nhiều traffic khi team product thêm field mới. Vì vậy tôi chọn schema lõi bắt buộc cộng với cột `metadata` linh hoạt. Dòng lỗi đi vào quarantine thay vì làm dừng pipeline, vì mất một phần log tốt hơn làm hỏng toàn bộ batch.

## 2. Batch hay streaming

Tôi chọn mô hình lai: streaming nhẹ cho monitoring gần real-time, batch hằng đêm cho dataset training. Streaming giúp phát hiện spike về lỗi tool, hallucination hoặc latency trong vài phút, nhưng nếu dùng streaming cho toàn bộ logic dataset thì phức tạp, khó backfill và dễ sai decontamination. Batch hằng đêm rẻ hơn, dễ kiểm tra lại, và phù hợp vì fine-tuning không cần cập nhật từng phút. Tradeoff là eval/training dataset chậm hơn vài giờ, nhưng đổi lại pipeline dễ audit và tái lập.

## 3. Quality gate và PII

Tôi sẽ validate trước khi dữ liệu vào tầng Silver: timestamp hợp lệ, trace tree không bị đứt, span con có parent đúng, tool result không rỗng với tool bắt buộc, và nội dung đã được redact PII. Với thị trường Việt Nam, số điện thoại, địa chỉ, mã vận đơn và tài khoản ngân hàng rất dễ xuất hiện trong chat. Tôi chọn redaction trước khi tạo eval/DPO, dù có thể làm mất một ít ngữ cảnh. Tradeoff là bảo vệ riêng tư quan trọng hơn giữ nguyên câu chữ. Khi quarantine tăng đột biến, pipeline phải gửi alert cho người vận hành vì đó có thể là lỗi instrumentation từ app, không phải lỗi model.

## 4. Eval, DPO và decontamination

Eval set phải được giữ riêng theo thời điểm và chủ đề. Tôi sẽ lấy các conversation đã được human review hoặc có feedback rõ ràng làm candidate, sau đó sample cân bằng giữa đổi trả, hoàn tiền, bảo hành, voucher và giao hàng. DPO pairs được tạo từ các prompt có cả phản hồi tốt và phản hồi lỗi, hoặc từ human correction. Tradeoff là dùng toàn bộ traffic sẽ có nhiều dữ liệu hơn, nhưng nhiễu hơn và dễ chứa prompt nằm trong eval. Tôi chọn dữ liệu ít hơn nhưng sạch hơn. Decontamination là bắt buộc: nếu prompt trong eval cũng xuất hiện trong DPO train, điểm benchmark sẽ tăng giả vì model học thuộc tình huống kiểm tra.

## 5. Point-in-time feature

Một feature nguy hiểm là `refund_count_30d` hoặc `total_spend_before_ticket`. Nếu ticket xảy ra lúc 10:00 nhưng pipeline join nhầm trạng thái đơn hàng lúc 18:00, model sẽ biết trước việc hoàn tiền đã được duyệt. Tôi chọn ASOF join theo timestamp của conversation. Tradeoff là PIT feature phức tạp hơn join mới nhất, nhưng cần thiết để tránh leakage. Đặc biệt với fraud hoặc refund abuse, leakage làm model offline có vẻ rất giỏi nhưng production lại đoán sai khi chưa có sự kiện tương lai.

## 6. Vector hay knowledge graph

Vector index phù hợp cho câu hỏi trực tiếp như “đổi trả trong bao nhiêu ngày” vì chỉ cần tìm đoạn chính sách liên quan. Knowledge graph phù hợp cho câu multi-hop như “sản phẩm thuộc nhóm điện tử, mua trong chiến dịch sale, đã mở hộp thì có được hoàn tiền hay chỉ bảo hành không”. Tôi chọn dùng cả hai: vector làm retrieval mặc định, KG dùng cho policy có quan hệ điều kiện. Tradeoff là KG tốn công extract và maintain, nên không dùng cho mọi tài liệu. Các rule nhiều điều kiện, có ngoại lệ và quan hệ giữa sản phẩm, chính sách, chiến dịch mới đáng đưa vào graph.

## Phương án bị loại

Tôi loại phương án “đẩy toàn bộ log production trực tiếp vào training mỗi tuần”. Cách này rẻ và nhanh lúc đầu, nhưng nguy hiểm vì ba lý do: chứa PII, chứa câu trả lời sai của model, và trộn lẫn eval với train. Khi model học lại lỗi cũ, flywheel trở thành vòng lặp tự đầu độc. Tôi cũng không chọn chỉ dùng vector RAG mà bỏ trace analytics, vì như vậy chatbot có thể trả lời được một số câu chính sách nhưng team không biết lỗi đang đến từ retrieval, tool, prompt hay model.

## Vận hành và chi phí

Chi phí lớn nhất nằm ở lưu trace dài, embedding tài liệu, và human review. Tôi sẽ cắt chi phí bằng sampling có chủ đích: giữ toàn bộ trace lỗi, trace có escalation, trace có feedback xấu; còn traffic thành công thì sample theo chủ đề. Embedding chỉ chạy lại với tài liệu thay đổi, không rebuild toàn bộ index mỗi ngày. Backfill phải idempotent theo `trace_id` và `dataset_version`, để chạy lại không nhân đôi preference pair. Dataset dùng cho train phải có version rõ ràng: nguồn trace nào, ngày nào, rule decontamination nào, và eval set nào bị loại. Với pipeline AI, khả năng giải thích dataset quan trọng gần bằng bản thân model.

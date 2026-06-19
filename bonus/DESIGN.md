# Bonus Design — Flywheel dữ liệu cho chatbot CSKH tiếng Việt

## Bài toán

Bài toán tôi chọn là pipeline dữ liệu cho một chatbot chăm sóc khách hàng tiếng Việt. Chatbot nhận câu hỏi từ người dùng về đổi trả, bảo hành, giao hàng, hóa đơn và khiếu nại sản phẩm. Mỗi lượt chat sinh ra nhiều loại dữ liệu: câu hỏi gốc, câu trả lời của agent, tài liệu được retrieve, tool call, lỗi tool, đánh giá của người dùng, và thao tác sửa của nhân viên CSKH.

Mục tiêu không chỉ là lưu log. Mục tiêu là biến traffic thật thành một vòng lặp học được: trace sản xuất đi vào Bronze, được lọc lỗi, tách thành eval set, tạo preference pairs, rồi dùng cho cải thiện prompt, RAG, hoặc DPO/SFT sau này. Nếu làm sai, hệ thống có thể tự học từ câu trả lời sai của chính nó. Vì vậy pipeline phải coi chất lượng dữ liệu là phần lõi, không phải bước phụ.

Ràng buộc thực tế là tiếng Việt có dấu và không dấu, người dùng viết tắt, sai chính tả, trộn tiếng Anh, gửi ảnh chụp hóa đơn, và nhiều câu hỏi không đủ ngữ cảnh. Dữ liệu cũng có thể chứa số điện thoại, địa chỉ, mã đơn hàng. Pipeline phải rẻ, chạy được với đội nhỏ, và không được để dữ liệu nhạy cảm lọt vào tập train thô.

## Sơ đồ kiến trúc

```text
User chat / CSKH inbox
        |
        v
Trace collector
        |
        v
Bronze trace store  -----> PII scrubber -----> Quarantine / review queue
        |
        v
Silver normalized turns
        |
        +--> Eval curator --------> eval_golden.jsonl
        |
        +--> Preference miner ----> raw pairs ----> decontamination ----> preference_pairs.jsonl
        |
        +--> RAG/KG feedback -----> missing-doc tickets / doc update backlog
        |
        v
Gold monitoring tables
  - failure rate by intent
  - retrieval miss rate
  - low-confidence answer rate
  - human correction rate
```

## 1. Nguồn và hình dạng dữ liệu

Nguồn chính là trace từ chatbot và thao tác sửa của nhân viên. Tôi sẽ không chỉ lưu message cuối cùng, vì message cuối không giải thích vì sao agent trả lời sai. Cần lưu cây span: retrieve, rerank, tool call, LLM answer, guardrail, và final response.

Đánh đổi là lưu trace đầy đủ tốn dung lượng hơn so với chỉ lưu transcript. Tôi vẫn chọn trace đầy đủ vì lỗi RAG thường nằm ở giữa pipeline: retrieve sai tài liệu, tool timeout, hoặc guardrail sửa câu trả lời. Nếu chỉ giữ transcript, đội vận hành chỉ thấy kết quả sai nhưng không thấy nguyên nhân.

Dữ liệu thô đi vào Bronze theo kiểu append-only. Silver mới chuẩn hóa schema, mask PII, chuẩn hóa intent, và gắn nhãn outcome. Dòng lỗi không bị xóa; nó đi vào quarantine để điều tra.

## 2. Batch hay streaming

Tôi chọn micro-batch mỗi 5–15 phút cho phần cải thiện dữ liệu, không chọn streaming từng event cho toàn bộ pipeline. Chatbot cần trả lời người dùng real-time, nhưng eval set và DPO pairs không cần cập nhật từng giây.

Tradeoff: streaming cho dashboard nhanh hơn, nhưng phức tạp hơn về idempotency, checkpoint, replay và cost. Micro-batch đủ tốt để phát hiện spike trong ngày và rẻ hơn cho đội nhỏ. Riêng alert nghiêm trọng như nhiều câu trả lời sai về hoàn tiền hoặc khiếu nại pháp lý thì xử lý streaming nhẹ: chỉ đẩy counter và cảnh báo, không chạy toàn bộ curation pipeline ngay lúc đó.

## 3. Hợp đồng dữ liệu và quality gate

Trước khi dữ liệu vào tập train, tôi validate tối thiểu các trường: `trace_id`, `turn_id`, `timestamp`, `user_input`, `agent_output`, `outcome`, `retrieved_doc_ids`, `latency_ms`, `cost`, và `human_feedback` nếu có. Một trace thiếu `user_input` hoặc mất `trace_id` phải bị quarantine, vì nó không thể ghép lại thành eval row hay preference pair an toàn.

Tradeoff là gate quá chặt có thể loại nhiều dữ liệu hữu ích. Nhưng với dữ liệu dùng để train/eval, tôi ưu tiên ít nhưng sạch. Log thô vẫn giữ trong Bronze để debug; chỉ Silver/Gold mới yêu cầu schema chặt. Khi quarantine tăng đột ngột, đó có thể là lỗi instrumentation chứ không phải lỗi người dùng, nên cần alert cho người phụ trách pipeline.

## 4. Train/serve parity và point-in-time

Một lỗi nguy hiểm là dùng thông tin biết sau thời điểm chat để train model. Ví dụ, sau khi nhân viên sửa câu trả lời, pipeline gắn nhãn rằng câu trả lời ban đầu sai. Nếu lúc train ta join nhãn này như một feature đầu vào, model sẽ thấy tương lai.

Tôi sẽ tách rõ feature tại thời điểm phục vụ và label sau phục vụ. Feature phục vụ gồm intent dự đoán, tài liệu được retrieve lúc đó, confidence, và metadata sản phẩm có hiệu lực tại thời điểm chat. Label sau phục vụ gồm rating, human correction, refund outcome. Với bảng thay đổi theo thời gian như policy đổi trả, phải dùng point-in-time join để agent không học từ version chính sách chưa tồn tại ở thời điểm câu hỏi.

Tradeoff là point-in-time table phức tạp hơn bảng latest snapshot. Tôi chọn point-in-time vì chatbot CSKH dễ gặp lỗi chính sách: nếu chính sách đổi trả thay đổi ngày 15, không thể dùng policy ngày 20 để đánh giá câu trả lời ngày 10.

## 5. RAG hay knowledge graph

Vector RAG phù hợp với câu hỏi lookup: “bảo hành gadget bao lâu”, “đổi trả trong mấy ngày”. Knowledge graph phù hợp với câu hỏi nhiều bước: “sản phẩm này thuộc bộ phụ kiện nào, kho nào xử lý, chính sách đổi trả áp dụng theo nhóm nào”.

Tôi chọn hybrid. Vector RAG là mặc định vì rẻ, dễ update, và đủ cho phần lớn FAQ. Knowledge graph chỉ dùng cho nhóm câu hỏi cần quan hệ rõ: sản phẩm → nhóm sản phẩm → chính sách → ngoại lệ → kho xử lý. Tradeoff là KG tốn công chuẩn hóa entity và relation. Nếu ép mọi thứ vào KG, tốc độ phát triển chậm. Nếu chỉ dùng vector, các câu hỏi multi-hop dễ trả lời thiếu một bước.

## 6. Flywheel và decontamination

Pipeline sẽ tạo eval set từ các lượt đã được con người xác nhận đúng hoặc sửa thành đúng. Preference pairs được tạo bằng cách ghép câu trả lời tốt với câu trả lời lỗi cho cùng hoặc rất gần cùng một prompt.

Bước bắt buộc là decontamination. Nếu prompt đã nằm trong eval set, nó không được nằm trong tập train/DPO. Nếu bỏ bước này, eval sẽ báo điểm cao giả vì model đã học đúng câu hỏi đó. Với tiếng Việt, exact match là chưa đủ. Người dùng có thể viết “đổi trả widget 10 ngày được không” và “mua widget 10 hôm rồi trả được không”. Vì vậy bản production nên thêm fuzzy matching bằng n-gram hoặc embedding similarity.

Tradeoff là fuzzy decontamination có thể loại nhầm vài pair hợp lệ. Tôi chấp nhận mất một ít dữ liệu train để bảo vệ độ tin cậy của eval. Eval sạch quan trọng hơn số lượng pair lớn nhưng nhiễm.

## Phương án bị loại

Tôi loại phương án “train trực tiếp trên toàn bộ transcript production”. Lý do: transcript chứa câu trả lời sai, PII, prompt nằm trong eval, và các lỗi tạm thời do tool timeout. Nếu train thẳng, hệ thống tự khuếch đại lỗi của chính nó. Phương án đúng hơn là Bronze giữ tất cả, Silver lọc và chuẩn hóa, rồi chỉ những turn đủ điều kiện mới đi vào eval hoặc preference dataset.

## Quyết định cuối

Thiết kế tôi chọn là micro-batch flywheel với Bronze append-only, Silver quality gate, eval/decontamination rõ ràng, và hybrid RAG/KG. Pipeline này không cố làm mọi thứ real-time. Nó ưu tiên ba điểm: dữ liệu dùng để học phải sạch, eval không được bị leak, và lỗi phải truy vết được từ câu trả lời cuối về trace gốc. Với chatbot tiếng Việt, đây là cách thực tế hơn so với một pipeline lớn nhưng khó vận hành và dễ tự học sai.

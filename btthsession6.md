# THIẾT KẾ GIẢI PHÁP PROMPT CHAINING: TỰ ĐỘNG HÓA CHẤM RUBRIC & GIA SƯ FEYNMAN KHÉP KÍN



## PHẦN 1: BÁO CÁO THIẾT KẾ GIẢI PHÁP (SOLUTION DESIGN REPORT)

### 1. Bối cảnh và Bài toán thực tế

Trong đào tạo nội bộ và đánh giá năng lực học tập tại **TechShop**, học viên/nhân sự mới cần nắm vững các quy định nghiệp vụ và nguyên lý kỹ thuật thông qua việc diễn giải lại kiến thức bằng ngôn ngữ đơn giản (phương pháp Feynman). Nếu chỉ dùng một câu lệnh đơn lẻ (Single Prompt) để vừa kiểm tra dữ liệu lỗi, vừa chấm điểm theo Rubric, vừa phân tích lỗ hổng và đóng vai gia sư giảng lại, mô hình dễ bị dàn trải suy luận, tính điểm thiếu ổn định hoặc xử lý thiếu nhất quán khi gặp bài làm rỗng/sai lệch.

Giải pháp được thiết kế dưới dạng **chuỗi Prompt Chaining chạy tuần tự khép kín gồm 3 bước**, có truyền dữ liệu có cấu trúc (Data Handoff) và cơ chế bắt lỗi dị biệt (Guardrail & Fallback) để hệ thống luôn trả về kết quả sạch lỗi, không bị ngắt mạch hay crash luồng xử lý.

### 2. Kiến trúc Luồng Dữ liệu Khép kín (Closed-Loop Chaining Architecture)

```text
[Dữ liệu đầu vào: Quy định/Chủ đề chuẩn + Rubric + Bài giải thích của học viên]
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  BƯỚC 1: Kiểm tra Dữ liệu Dị biệt & Chấm điểm Rubric (Validation & Score) │
│  - Lọc bài trống, vô nghĩa, lạc đề hoặc chứa dữ liệu sai lệch rõ ràng.   │
│  - Chấm từng tiêu chí theo Rubric & cộng tổng tuần tự.                    │
└──────────────────────────────────────────────────────────────────────────┘
                                       │
                        (Truyền JSON Handoff sạch lỗi)
                                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  BƯỚC 2: Bóc tách Lỗ hổng Kiến thức (Feynman Gap Diagnostic)            │
│  - Đối chiếu điểm trừ để xác định chính xác lỗ hổng hiểu biết.          │
│  - Phân loại mức độ (Chưa nắm khái niệm / Hiểu sai quy tắc / Thiếu ý).  │
└──────────────────────────────────────────────────────────────────────────┘
                                       │
                        (Truyền JSON Handoff chẩn đoán)
                                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  BƯỚC 3: Phản hồi Gia sư Feynman & Câu hỏi Kiểm chứng (Tutor Loop)       │
│  - Giải thích lại chỗ hổng bằng ví dụ đời thường ngắn gọn, dễ hiểu.     │
│  - Giao 1 câu hỏi tự kiểm tra để khép kín chu trình học tập.            │
└──────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
[Đầu ra hoàn chỉnh: JSON trạng thái/điểm + Chẩn đoán lỗ hổng + Lời gia sư]

```

### 3. Cơ chế Tối ưu Token & Chống Lỗi Dị biệt

* **Tối ưu token:** Mỗi prompt chỉ giữ đúng ngữ cảnh và đầu vào cần thiết cho khâu đó, dùng JSON gọn trường và không lặp lại toàn bộ hướng dẫn dài dòng ở các bước sau.
* **Bắt lỗi dữ liệu dị biệt (Anomaly Guardrail):** Nếu bài giải thích trống, chỉ có ký tự vô nghĩa, lạc đề hoặc sai quy định chuẩn, Bước 1 gán `status` rõ ràng (`INVALID_INPUT` hoặc `NEEDS_CORRECTION`) và ghi nhận `error_flags` / `is_valid: false`, giúp các bước sau chuyển sang chế độ xử lý an toàn mà không bịa thêm dữ liệu.

---

## PHẦN 2: HỆ THỐNG PROMPT THỰC THI CÁC BƯỚC (PROMPT CHAINING)

###  Bước 1: Prompt Kiểm tra Dữ liệu & Chấm Điểm Rubric

```text
[Role]
Bạn là bộ chấm điểm chuẩn theo Rubric tại TechShop.

[Task]
Kiểm tra tính hợp lệ của `learner_explanation` so với `topic_rule`, sau đó chấm điểm từng tiêu chí trong `rubric` và tính tổng điểm chính xác.

[Input]
- topic_rule: "Khi khách nhận hàng lỗi kỹ thuật hoặc móp méo do vận chuyển tại TechShop, nhân viên cần xác nhận mã đơn hàng, xin lỗi chân thành, đổi mới 100% trong 24h (miễn phí vận chuyển 2 chiều) nếu đủ điều kiện, và tuyệt đối không tự ý bịa mã đơn khi khách chưa cung cấp."
- rubric:
  + accuracy (max 4.0): Đúng quy định chuẩn (24h, miễn phí ship 2 chiều nếu đủ điều kiện, không bịa mã đơn).
  + feynman_clarity (max 3.0): Giải thích đơn giản, rõ ràng, dễ hiểu.
  + exception_handling (max 3.0): Xử lý đúng tình huống thiếu mã đơn / ngoại lệ.
- learner_explanation: "Nếu khách bị lỗi máy thì mình đổi mới trong 3 ngày, còn thiếu mã đơn thì em điền đại #TS-0000 cho nhanh để hệ thống khỏi báo lỗi."

[Rules & Error Handling]
- Nếu `learner_explanation` rỗng, chỉ chứa ký tự vô nghĩa hoặc hoàn toàn không liên quan đến `topic_rule`: đặt `status = "INVALID_INPUT"`, `is_valid = false`, cho 0 điểm tất cả mục, ghi rõ lý do vào `error_flags`. Không tự suy diễn nội dung thay người học.
- Nếu có thông tin sai lệch hoặc thiếu sót so với `topic_rule`: đặt `status = "NEEDS_CORRECTION"`, `is_valid = true`, ghi lỗi vào `error_flags`. Không tự sửa câu trả lời của người học thành đúng.
- Nếu đầy đủ và đúng chuẩn: đặt `status = "PASSED"`, `is_valid = true`, `error_flags = []`.
- Cộng điểm từng mục rõ ràng, không làm tròn lệch. Chỉ xuất JSON thuần túy.

[Output JSON]
{
  "status": "...",
  "is_valid": true,
  "error_flags": [],
  "scores": {
    "accuracy": 0,
    "feynman_clarity": 0,
    "exception_handling": 0,
    "total": 0
  },
  "learner_explanation": "..."
}

```

---

###  Bước 2: Prompt Chẩn đoán Lỗ hổng Kiến thức (Feynman Diagnostic)

```text
[Role]
Bạn là chuyên gia phân tích lỗ hổng học tập theo phương pháp Feynman.

[Input From Step 1]
[Dán JSON đầu ra của Bước 1 vào đây]

[Task]
Dựa trên `status`, `is_valid`, `error_flags`, `scores` và `learner_explanation` từ Bước 1, xác định các lỗ hổng kiến thức cụ thể để chuyển tiếp cho gia sư.

[Rules & Error Handling]
- Nếu `is_valid == false` hoặc `status == "INVALID_INPUT"`: không chẩn đoán chuyên môn, trả về `gaps: []` và `fallback_note: "Thiếu hoặc sai dữ liệu đầu vào hợp lệ, cần nhập lại phần giải thích đúng chủ đề."`.
- Nếu `status == "PASSED"`: trả về `gaps: []` và `fallback_note: null`.
- Nếu `status == "NEEDS_CORRECTION"`: liệt kê ngắn gọn từng lỗ hổng (`concept`, `learner_claim`, `standard_rule`), không thêm thông tin ngoài dữ liệu đầu vào. Chỉ xuất JSON thuần túy.

[Output JSON]
{
  "status": "...",
  "is_valid": true,
  "fallback_note": null,
  "gaps": [
    {
      "concept": "...",
      "learner_claim": "...",
      "standard_rule": "..."
    }
  ]
}

```

---

###  Bước 3: Prompt Phản hồi Gia sư Feynman & Câu hỏi Kiểm chứng

```text
[Role]
Bạn là gia sư Feynman tận tâm, rõ ràng, dùng ngôn ngữ đời thường gần gũi.

[Input From Step 2]
[Dán JSON đầu ra của Bước 2 vào đây]

[Task]
Tạo phản hồi gia sư ngắn gọn dựa trên đầu ra của Bước 2:
- Nếu có `fallback_note` không rỗng: giải thích nhẹ nhàng lý do chưa thể đánh giá nội dung, hướng dẫn cách viết lại 1-2 câu đơn giản đúng chủ đề và đặt 1 câu hỏi gợi mở.
- Nếu `status == "PASSED"`: ghi nhận điểm mạnh của cách diễn đạt dễ hiểu và đặt 1 câu hỏi tình huống ngắn để củng cố.
- Nếu `status == "NEEDS_CORRECTION"`: giải thích lại từng `gap` bằng ví dụ đời thường dễ hình dung, tránh thuật ngữ rườm rà, sau đó đặt 1 câu hỏi tự kiểm tra ngắn.

[Constraints]
- Không bịa thêm quy định ngoài `standard_rule`.
- Giữ giọng điệu khách quan, hỗ trợ, súc tích.

```

---

## PHẦN 3: KẾT QUẢ ĐẦU RA SẠCH LỖI CỦA AI (CLEAN OUTPUT SIMULATION)

### 1. Kết quả Bước 1 (Kiểm tra & Chấm Rubric)

```json
{
  "status": "NEEDS_CORRECTION",
  "is_valid": true,
  "error_flags": [
    "Sai thời hạn đổi mới: nêu 3 ngày thay vì 24h theo quy định chuẩn",
    "Sai cách xử lý khi thiếu mã đơn: tự điền mã giả #TS-0000 thay vì không tự bịa mã đơn"
  ],
  "scores": {
    "accuracy": 1.5,
    "feynman_clarity": 2.0,
    "exception_handling": 0.0,
    "total": 3.5
  },
  "learner_explanation": "Nếu khách bị lỗi máy thì mình đổi mới trong 3 ngày, còn thiếu mã đơn thì em điền đại #TS-0000 cho nhanh để hệ thống khỏi báo lỗi."
}

```

### 2. Kết quả Bước 2 (Chẩn đoán Lỗ hổng)

```json
{
  "status": "NEEDS_CORRECTION",
  "is_valid": true,
  "fallback_note": null,
  "gaps": [
    {
      "concept": "Thời hạn đổi mới hàng lỗi/móp méo",
      "learner_claim": "Đổi mới trong 3 ngày",
      "standard_rule": "Đổi mới 100% trong 24h (miễn phí vận chuyển 2 chiều nếu đủ điều kiện)"
    },
    {
      "concept": "Xử lý khi khách chưa cung cấp mã đơn hàng",
      "learner_claim": "Tự điền đại mã #TS-0000 cho nhanh",
      "standard_rule": "Tuyệt đối không tự ý bịa mã đơn khi khách chưa cung cấp"
    }
  ]
}

```

### 3. Kết quả Bước 3 (Phản hồi Gia sư Feynman)

Cách diễn đạt của bạn khá thẳng thắn và dễ nắm ý, nhưng hiện có **2 điểm cần chỉnh lại** để khớp với quy định của TechShop:

* **Mốc thời gian đổi mới là 24h, không phải 3 ngày:** Quy định chuẩn là đổi mới 100% trong **24h** (kèm miễn phí vận chuyển 2 chiều nếu đủ điều kiện). Nếu nói 3 ngày, mốc thời gian cam kết sẽ bị lệch so với chuẩn.
* **Không tự điền mã đơn giả (`#TS-0000`):** Hãy hình dung sổ theo dõi đơn hàng giống như bảng phân ghế trên chuyến xe. Nếu khách chưa báo số ghế mà bạn tự ghi bừa một số có sẵn, lúc đối soát sẽ dễ khớp nhầm sang khách khác hoặc làm sai lệch dữ liệu. Khi thiếu mã đơn, cần giữ nguyên tắc không tự bịa mã và xác minh thông tin chuẩn từ khách.

**Câu hỏi tự kiểm tra:** Nếu khách nhắn báo máy lỗi nhưng chưa cung cấp mã đơn hàng, bạn sẽ xử lý bước đầu thế nào để vừa giữ thái độ hỗ trợ tốt vừa không tạo mã giả?
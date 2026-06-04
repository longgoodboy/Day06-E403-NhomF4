# SPEC Sản Phẩm — Vietnam Airlines NEO AI Triage

## 1. Bằng chứng

Nỗi đau mà nhóm muốn giải quyết xuất phát từ trải nghiệm hỗ trợ khách hàng sau giao dịch thanh toán (vé hoặc hành lý mua thêm).

- **Trải nghiệm trực tiếp (Self-use evidence):** Nhóm đã đóng vai hành khách sử dụng NEO chatbot của Vietnam Airlines. Qua các tình huống thử nghiệm (xem ảnh chụp màn hình trong `02-group-spec/evidence`), nhóm nhận thấy:
  - Khi user đặt câu hỏi phức tạp (chứa nhiều vấn đề), bot trả lời đoạn văn bản rất dài nhưng chưa bóc tách rõ từng ý (multi-intent) để giải quyết dứt điểm.
  - Với tình huống rủi ro cao (ví dụ: "app báo lỗi nhưng tiền đã trừ"), bot vẫn trả về thông tin chính sách chung chung thay vì có một luồng khẩn cấp.
  - Khi bot cần thông tin, bot hỏi rất chung chung thay vì đưa ra một danh sách (checklist) rõ ràng những gì hành khách cần cung cấp.
  - Bot dễ bị quên bối cảnh (context loss) khi người dùng đính chính.
- **Nguồn từ bên ngoài (Analog/Competitor evidence):** Các app ngân hàng hoặc e-commerce khi xử lý khiếu nại (dispute support) đều có luồng Triage (phân loại) rất rõ ràng: định tuyến vấn đề -> hỏi checklist thông tin bắt buộc -> đánh dấu rủi ro -> chuyển tư vấn viên.
- **Nhận định (Giả định):** Khi khách hàng mất tiền mà chưa nhận được vé/dịch vụ, họ rơi vào trạng thái hoang mang tột độ. Việc chatbot trả lời chính sách dài dòng làm tăng sự ức chế.

## 2. Lát cắt để build

Thay vì làm toàn bộ luồng hỗ trợ, nhóm tập trung vào lát cắt nhỏ nhất:
**AI Triage (Phân loại tự động) cho hành khách gặp sự cố sau khi thanh toán vé hoặc mua thêm hành lý.** 
Một người dùng báo lỗi thanh toán -> AI quyết định bóc tách loại lỗi, đánh giá mức độ rủi ro, thu thập đủ thông tin -> Kết quả trả về là một bảng tóm tắt (Handoff Summary) hoàn chỉnh để chuyển cho tư vấn viên con người xử lý.

## 3. AI Product Canvas

| Ô | Câu trả lời |
| --- | --- |
| **Value (Giá trị)** | **Dành cho ai:** Hành khách gặp lỗi sau giao dịch thanh toán trực tuyến.<br>**Đau ở đâu:** Hoang mang vì tiền đã trừ nhưng dịch vụ chưa được xác nhận, chat với bot thì bị hỏi vòng vo.<br>**AI giải quyết gì:** Phân loại đúng vấn đề ngay lập tức, đưa ra đúng checklist cần chuẩn bị và gói gọn context chuyển cho nhân viên. |
| **Trust (Niềm tin)** | **Khi sai:** Nếu AI nhận diện nhầm (từ vé sang hành lý), người dùng dễ dàng đính chính lại bằng text ("Không, tôi hỏi hành lý"). Bot sẽ lập tức xin lỗi, chuyển intent mà vẫn nhớ bối cảnh "tiền đã trừ".<br>**Chuyển giao:** Luôn hiển thị trạng thái rủi ro và có nút "Chuyển nhân viên" đi kèm bản tóm tắt thông tin. |
| **Feasibility (Khả thi)** | Rất đáng build vì giới hạn chỉ 3 intent chính (vé, hành lý, thanh toán chung). Rủi ro thấp do AI không tự ý hoàn tiền hay xuất vé (chỉ đóng vai trò thu thập thông tin). API chạy tốn ít token, latency phản hồi nhanh. Ngưỡng dừng: Nếu API timeout, tự động fallback về luồng mock mặc định. |
| **Tín hiệu học** | Tín hiệu đến từ các lần người dùng phải "sửa lưng" bot (Correction path) hoặc bỏ ngang để bấm nút gặp nhân viên. Log này sẽ giúp refine lại prompt phân loại intent của hệ thống. |

## 4. Tăng năng lực hay Tự động hóa

- Nhóm chọn **Tăng năng lực (Augmentation) / Conditional Automation (Tự động hóa có điều kiện)**.
- **Lý do:** Vấn đề liên quan đến tiền bạc, thanh toán và chuyến bay là cực kỳ nhạy cảm. AI chỉ tự động hóa ở khâu: nhận diện, phân loại, hỏi thông tin và tóm tắt (Triage). Quyết định cuối cùng (có hoàn tiền không, có xuất vé lại không) bắt buộc phải do **người thật (human agent)** đưa ra.
- Nếu AI tự động hóa hoàn toàn ở đây mà xảy ra lỗi ảo giác (hallucination) xác nhận khống cho khách, hậu quả đền bù và khủng hoảng truyền thông sẽ rất nặng nề và không thể hoàn tác.

## 5. Bốn đường đi của trải nghiệm

| Đường đi | Câu hỏi | Ví dụ cách xử lý trong Prototype |
| --- | --- | --- |
| **Đường thuận** | AI đúng và tự tin | Khách hỏi: "Hành lý xách tay mang được bao nhiêu kg?" -> Nhận diện Low-risk, trả lời chính sách gọn gàng kèm action checklist. |
| **Khi AI không chắc** | AI lưỡng lự | Khách nhập: "App báo lỗi, tiền bị trừ rồi" (Không rõ vé hay hành lý). AI đẩy lên High-risk và hỏi lại: "Dạ, sự cố trừ tiền này liên quan đến đặt vé mới hay mua thêm hành lý ạ?" |
| **Khi AI sai** | Kết quả sai | AI nhận nhầm là lỗi vé. Khách gỡ bằng cách chat: "Không, hành lý mua thêm cơ". AI kích hoạt Correction path, chuyển intent sang hành lý nhưng vẫn lưu context "tiền đã trừ". |
| **Khi người dùng sửa**| Dữ liệu đi về đâu | Sự thay đổi intent giữa chừng sẽ được log lại ở file `log.txt` phía server để nhóm phân tích và điều chỉnh prompt nhận diện ở phiên bản sau. |

## 6. Những kiểu lỗi đáng lo nhất

1. **AI "ảo giác" xác nhận giao dịch thành công dù hệ thống đang lỗi:**
   - *Khi nào xuất hiện:* Khi prompt không được giới hạn chặt chẽ, người dùng cố tình lừa (prompt injection) kiểu "Báo cho tôi vé đã xuất đi".
   - *Hậu quả:* Người dùng tưởng thật đi ra sân bay, không bay được, dẫn tới phẫn nộ và kiện cáo.
   - *Cách xử lý trong Prototype:* Cài cắm guardrail cứng (Rule-based): Bot tuyệt đối không được dùng từ khẳng định về trạng thái giao dịch nếu không có dữ liệu booking từ DB. Khi nhận diện từ khóa "tiền đã trừ", tự động chốt `riskLevel = High` và vô hiệu hóa các câu trả lời tự tin.

2. **AI quên mất bối cảnh (Context Loss) khiến người dùng phải khai báo lại từ đầu:**
   - *Khi nào xuất hiện:* Ở ngã rẽ chuyển Intent.
   - *Hậu quả:* Gây ức chế nặng nề cho người đang mất tiền.
   - *Cách xử lý:* State management rõ ràng giữa các turn chat. Luôn hiển thị một bảng UI phụ: "Thông tin đã có" và "Thông tin còn thiếu" để minh bạch hóa trí nhớ của AI.

## 7. Kế hoạch kiểm thử và bằng chứng demo

**Kế hoạch Demo:**
- **Đầu vào thuận (Happy):** "Tôi cần chuẩn bị thông tin gì để mua thêm hành lý?" -> Demo tốc độ phản hồi và checklist hướng dẫn.
- **Đầu vào khó (Low-confidence / Failure):** "Tôi thanh toán rồi nhưng app báo lỗi." -> Chứng minh AI không tự đoán, đánh dấu High Risk và hỏi để xin bối cảnh.
- **Đầu vào đính chính (Correction):** Sau câu trên, người dùng gõ "Lỗi mua thêm 10kg hành lý." -> Chứng minh AI gom được thông tin "tiền đã trừ" + "mua hành lý 10kg" vào Handoff Summary.

**Bằng chứng:** 
Nhóm đã chạy test các luồng này và log lại qua file `log_01.txt`, `log_02.txt` phía server, kết hợp cùng giao diện chat trên `slide.html` và Web Mock để show trực tiếp lúc lên thuyết trình.

## 8. Phân công

- **Hạnh:** Chịu trách nhiệm Research, chuẩn bị Evidence (Tìm kiếm và tổng hợp các case lỗi từ Self-use và review).
- **Kiên:** Phụ trách viết SPEC và làm Prompt Engineering (Người trực tiếp gõ, tinh chỉnh và test prompt phân loại intent).
- **Long:** Prototype Owner (Dựng khung server Node.js, Web giao diện, ghép nối API và giữ repo GitHub).
- **Khoa:** Test Owner (Đảm bảo cover đủ 4 paths trải nghiệm, cố tình lừa bot bằng các prompt nhiễu để verify tính năng handoff).
- **Đức:** Chuẩn bị kịch bản Demo, điều phối nhịp độ thuyết trình và show bằng chứng.

# REPORT — Lab Day 12

| | |
|---|---|
| Họ và tên | `Dao Thi Huyen` |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `202602670` |
| GitHub | `halinhnhi123` |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | `2589 / 7668` |
| Số seed huấn luyện bạn dùng (`--seeds`) | `3` |

## Kết quả cuối

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---:|---:|---|
| constant-velocity | 1.26 | 19.70 | Không phát hành |
| IDM | 1.52 | 0.07 | (baseline) |
| EgoMLP | 0.76 | 18.98 | Không phát hành |
| PercMLP | 0.63 | 1.56 | Không phát hành |
| PercMLP+ (sau khi sửa) | 0.83 | 0.25 | Phát hành |

## Câu 1
Dữ liệu:
Bước 0 cho thấy dữ liệu có 9,993 quỹ đạo raw, sau khi loại trùng còn 9,610, tức 383 quỹ đạo bị bỏ; đây là cần thiết vì nếu để trùng lặp ở cả train/test thì mèo và chuột của cùng một kịch bản lặp lại sẽ làm đánh giá quá lạc quan. Cột `Acc_LV` không dùng vì hệ số tương quan với đạo hàm tốc độ chỉ là 0.045, quá yếu so với định nghĩa vật lý của gia tốc: `Acc_LV` là bản ghi nhiễu, còn gia tốc đáng tin cậy phải được suy từ `Speed_LV` bằng `np.gradient(...)` như trong `data.derive_lead`, và cả lab cũng dùng `a_lead` làm mạch phanh. Chia theo quỹ đạo, không chia theo khung, vì các khung của cùng một quỹ đạo rất gần nhau về động lực học và gây rò rỉ thông tin giữa train/test; do đó, giới hạn hợp lý là các quỹ đạo độc lập. Câu hỏi gợi ý này còn nhấn mạnh rằng ngưỡng phanh của xe dẫn phải được xác định từ gia tốc đã làm mượt, không từ cột `Acc_LV` thô.

## Câu 2
Phát lại:
Khi chạy lại cùng mã, `not_bit_identical = 0` và `differ_beyond_tol = 0`, chứng tỏ phát lại có tính tái lập; còn refactor float32 không giống từng bit (`not_bit_identical = 600`), nhưng mọi đoạn đều nằm trong dung sai `atol=1e-3, rtol=1e-3` với `max_abs_diff = 9.73e-05`, nên chênh lệch đó không quan trọng về hành vi. Tới thay đổi AEB, `differ_beyond_tol = 11` trên 600 đoạn, `not_bit_identical = 12`; các đoạn lệch xuất hiện quanh các khung mà `gap / closing < 1.5 s` khi `closing > 0`, tức khi xe dẫn đang tiến gần và TTC rất nhỏ. Ở đúng các khung đó, AEB chặn lệnh bằng `min(cmd, -6.0)`, nên các đoạn có vấn đề là những khung bắt đầu vào vùng nguy hiểm; đó là khoảng thời gian mà hành vi của planner thực sự đổi. Phát theo thời gian thật thì thấy `dropped = 573`, `late = 581` trên `frames = 59501`, trong khi phát nhanh nhất không có khung nào trễ hoặc rơi, cho thấy deadline không phải hư cấu: một khung chậm có thể làm lệnh trước đó bị giữ, khiến hệ thống “ngắt” và đẩy sai lệch rõ rệt khi chỉ áp dụng “nhanh nhất”.

## Câu 3
Kịch bản:
Lưới scenario dùng `v_ego = 10.0 m/s`, khoảng cách từ 2.76 m đến 59.31 m, và khoảng tốc độ tiến gần từ 0 đến 10 m/s, nên tổng là `45 x 60 = 2700` ô. Ở phản ứng `t_react = 0.6 s`, vùng không ai tránh được có `unavoidable_area = 0.051481...`, tức khoảng `5.15 %` của lưới; với `t_react = 0` thì vùng này chỉ `0.96296 %`, nên việc tăng độ trễ từ `0` lên `0.6 s` làm vùng “đâm không thể tránh được” tăng khoảng `4.19 điểm phần trăm`. Log có phủ khá tốt vùng khó: `mined_events = 614`, và phanh mạnh nhất của người lái trong log ở p99.9 là `3.62 m/s²`. Tỷ lệ ô trong lưới cần phanh mạnh hơn mức này là `5.22 %`, tương ứng với vùng mà mô hình học phải đối mặt nhưng log không luôn bao phủ; đồng thời, việc `full_brake_collision_area` khớp đúng với `unavoidable_area` cho thấy simulator là hợp lý: không có ô nào lệch xa biên vật lý. Đây là lý do vì sao ta cần test trên lưới kịch bản, không chỉ trên log.

## Câu 4
Vòng hở và vòng kín:
Bảng open-loop và closed-loop cho thấy sự đổi thứ tự rõ ràng: theo L2, `PercMLP` tốt nhất (`0.628`), tiếp theo `EgoMLP` (`0.765`), rồi `const-vel` (`1.260`), và `IDM` xấu nhất (`1.520`); nhưng trong vòng kín trên log, `IDM` lại được tốt nhất (`0.07 %` va chạm), `PercMLP` ở `1.56 %`, `EgoMLP` ở `18.98 %`, và `const-vel` ở `19.70 %`. Cặp đổi thứ tự đáng chú ý nhất là `IDM` vs `PercMLP`/`EgoMLP`: `IDM` có L2 tệ hơn `const-vel` nhưng lại đâm ít hơn nhiều vì L2 chỉ đo độ gần với hành vi con người trên trạng thái log, không đo mức độ an toàn trong điều khiển thực tế. `EgoMLP` tốt hơn `const-vel` về L2 (`0.765` vs `1.260`) nhưng vẫn đâm ở `18.98 %` vì nó không có cảm biến khoảng cách và tốc độ xe dẫn, nên không biết khi nào cần tác động nếu xe trước đang phanh hoặc cắt vào; nó chỉ nhìn “lịch sử của chính mình”, không “thế giới”. Việc này là ví dụ điển hình cho lý do mở rộng metric ra closed-loop và không tin vào điểm L2 đơn thuần.

## Câu 5
Một lần chạy chưa đủ:
EgoMLP có dao động lớn giữa các seed: `0.1779`, `0.1866`, `0.2049` tương ứng với các chạy seed 7668, 7669, 7670. Tính theo Wilson, khoảng tin cậy 95% của seed xấu nhất là `[0.1907, 0.2201]`, của seed tốt nhất `[0.1644, 0.1923]`; cả hai chồng lên nhau ở vùng 0.19–0.19 nhưng khoảng biến thiên `max-min = 0.027` (tương đương 2.7 điểm %) vẫn đủ lớn để quyết định kết luận. Nếu chỉ chạy một seed, bạn có thể báo `17.8 %` hoặc `20.5 %` tùy may rủi, dẫn đến kết luận sai về hiệu quả của mô hình; đây là lý do vì sao giảng viên đòi “độ rộng qua seed” và “phải coi dải va chạm” chứ không chỉ nhìn một lần chạy. Với `PercMLP`, dải qua seed chỉ khoảng `1.0..1.9 %`, nên mô hình này ổn định hơn, nhưng vẫn không đủ để bỏ qua kiểm tra trong thế giới phản ứng và trên lưới cut-in.

## Câu 6
Tác nhân phản ứng:
Khi hiệu chỉnh xe sau, khoảng cách `T` tối ưu được tìm là `1.2 s`; đây là giá trị tốt nhất vì nó làm hợp nhất với phân bố headway của người lái trong log và cho khoảng cách Wasserstein nhỏ nhất (`0.3495`) so với `never_reacting_agent_W1 = 0.8391`. Trong khi đó, xe sau không phản ứng hiện tượng “va chạm giả” rất rõ: với `IDM` ở `t_react = 0.6 s`, trong thế giới phát lại, `rear` chiếm `0.6663`, làm `any = 0.7178`; nhưng khi xe sau phản ứng thật, `rear = 0.0` và `any = 0.0515`. Tức là phần lớn va chạm phía sau trong bản phát lại là do xe sau bị “đóng băng” ở vận tốc log, không biết xe thử vừa phanh, nên nó không thể phản ứng; điều này làm `IDM` trông tệ hơn thực tế và giải thích vì sao trong môi trường phản ứng, các mô hình thật sự phải được đánh giá trên thế giới tương tác chứ không trên log cứng. Điểm mấu chốt là: mô hình có thể tốt trên log nhưng vẫn gặp khó ở kỳ vọng về vai trò của các xe lân cận.

## Câu 7
Cổng phát hành và sửa:
PercMLP bị chặn bởi nhiều quy tắc: `R2_closed_loop_logs` fail với upper 95 % = `2.5 %` vs baseline `0.1 % + 1.0 %` (vượt ngưỡng), `R3_braking_slice` fail vì `6.1 %` (upper `11.2 %`) so với baseline `0.0 % + 5 points`, `R4_scenario_grid` fail với `20.0 %` vs `0.0 % + 2.0 points`, và `R6_reactive_world` fail với `25.1 %` vs `5.1 %`. Đây là hai nguyên nhân gốc: thứ nhất, mô hình chỉ nhìn các trạng thái từ log, không có kịch bản cắt vào/xe dẫn phanh mà log không chứa; thứ hai, khi mô hình tự lái, sai số tích lũy khiến nó đi sâu vào vùng nguy hiểm. Sau khi sửa bằng `dagger_windows` + `synthetic cut-in scenarios`, tổng có `20000` kịch bản synthetic và `dagger_rounds = 2`, `decision_PercMLP+ = true`. `PercMLP+` vượt tất cả các quy tắc: `R2`: upper 95 % = `0.6 %` <= `0.1 % + 1.0 %`, `R3`: `0.0 %` (upper `2.5 %`), `R4`: `0.1 %` vs baseline `0.0 % + 2.0 %`, `R6`: `5.3 %` vs `5.1 %` vẫn pass bằng 0.2 điểm. Về mặt thực chất, `PercMLP+` tốt hơn `IDM` ở open-loop khi L2 giảm từ `1.52` xuống `0.83` nhưng vẫn không tốt hơn `IDM` trong đóng kín? Dù `0.25 %` va chạm thì thấp hơn `IDM` (`0.07 %`) ở mức trung bình nhưng vẫn có thể phát hành trong bối cảnh lab, vì nó đền bù được miền cut-in mà `IDM` hiện diện như baseline mẫu. Nếu cần cẩn trọng, tôi đề xuất chạy thêm 5–10 seed nữa và kiểm tra trên một tập vùng mới trước khi công bố ra sản phẩm thực tế.

## Câu 8
Bài lab này không chứng minh được gì:
Bài lab này chỉ cho thấy tập dữ liệu và kịch bản kiểm tra đang được xây dựng rất cẩn thận, không chứng minh mô hình sẽ an toàn trên mọi đường và mọi kiểu lái. Thứ nhất, dữ liệu đến từ Argoverse với các “car-following pair” còn hạn chế; không có nhiều tình huống chật hẹp, xe cắt vào bất ngờ, hay đa phương tiện. Thứ hai, đánh giá vẫn dựa trên mô phỏng 1-D rất giản hóa: vạch 1 chiều, không có lái lệch, không có người đi bộ, không có phản ứng của các phương tiện khác ngoài xe sau đơn giản. Thứ ba, chỉ có `3` seed huấn luyện và `2883` hành trình đánh giá trên log; con số này đủ để so sánh nhưng không đủ để khẳng định tính ổn định ở quy mô sản phẩm. Cuối cùng, lab không kiểm tra độ tin cậy trong môi trường thực tế: ảnh hưởng của nhiễu sensor, độ trễ controller, thay đổi đèn tín hiệu, và phân phối dữ liệu không ổn định. Để thu hẹp khoảng cách đến thế giới thật, bước tiếp theo cần là: tăng số lượng seed và vùng kiểm thử, chạy trên nhiều tuyến khác nhau, dùng nhiều kịch bản phản ứng hơn, và triển khai kiểm tra góc nhìn đa phương tiện với mô phỏng sát thực hơn.

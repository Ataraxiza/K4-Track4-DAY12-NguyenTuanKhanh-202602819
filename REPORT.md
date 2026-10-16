# REPORT — Lab Day 12

> Tôi điền các câu trả lời bằng số liệu thực tế đã xuất ra trong `evidence/*.json` của bài này. Tất cả số dùng ở đây đều xuất hiện trong bằng chứng đã có sẵn.

| | |
|---|---|
| Họ và tên | Nguyễn Tuấn Khanh |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `202602819` |
| GitHub | `https://github.com/Ataraxiza` |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | `split_seed=1958`, `model_seed=6963` |
| Số seed huấn luyện bạn dùng (`--seeds`) | `3` |

## Kết quả cuối

Các số dưới đây lấy từ `evidence/openloop.json`, `evidence/closedloop.json` và `evidence/gate.json`.

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---:|---:|---|
| constant-velocity | 1.2642 | 20.05% | Không qua |
| IDM | 1.5386 | 0.00% | (baseline) |
| EgoMLP | 0.7650 | 22.43% | Không qua |
| PercMLP | 0.6239 | 0.82% | Không qua |
| PercMLP+ (sau khi sửa) | 0.8258 | 0.23% | Qua |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột Acc_LV, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Bước 0 cho thấy `Acc_LV` không đáng tin cậy: hệ số tương quan giữa cột gia tốc xe dẫn và đạo hàm tốc độ xe dẫn chỉ là `0.04535`, trong khi cột tương ứng của xe theo sau là `0.92909`. Điều này nghĩa là `Acc_LV` không khớp với `d(Speed_LV)/dt`; nên ta dùng gia tốc đã làm mượt từ `Speed_LV` thay vì cột nguyên thủy. Dữ liệu có `383` quỹ đạo trùng chính xác trong tổng `9993` quỹ đạo raw; nếu chia theo khung hình thay vì theo quỹ đạo, các chuỗi giống nhau sẽ rò rỉ giữa train/test và đánh giá sai. Ta chia theo quỹ đạo vì mỗi quỹ đạo là một đơn vị động lực riêng, còn các frame trong cùng quỹ đạo là cùng một chuỗi thời gian không độc lập.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Kết quả replay cho thấy refactor float32 không giống từng bit: `not_bit_identical = 600`, nhưng `differ_beyond_tol = 0`. Nghĩa là mọi đoạn thay đổi bit nhưng không vượt ngưỡng dung sai `atol=1e-3`, `rtol=1e-3`, nên nó vẫn là cùng một hành vi khi xét theo độ lệch cho phép. Với AEB, số đoạn bị lệch vượt dung sai là `11` trên `600`, `not_bit_identical = 12`, max abs diff khoảng `1.4356`; sự lệch xảy ra đúng ở các khung có TTC thấp và khoảng cách đang tiến gần, vì lúc đó `min(cmd, -6.0)` mới kích hoạt và làm lệnh vượt ngưỡng theo cấu hình AEB. Phát theo thời gian thật cho thấy `dropped = 580`, `late = 582`, trong khi phát nhanh nhất có `dropped = 0`, `late = 0`; nghĩa là phát nhanh nhất che đi độ trễ xử lý và khung bị rớt, nên nó không phản ánh thời gian hệ thống thực sự xử lý sự kiện.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô không ai tránh được? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới cắt vào của tôi có `v_ego=12.0 m/s`, khoảng cách từ `2.76 m` đến `59.31 m`, và `dv` từ `0` đến `12.0 m/s`; tổng số ô là `2700` và vùng không ai tránh được ở `t_react=0.0` là `0.02222` = `2.22%`, còn ở `t_react=0.6` là `0.07444` = `7.44%`. Tức là khi trễ tăng từ 0 lên 0,6s, vùng va chạm bất khả thi tăng khoảng `5.22` điểm phần trăm, và đường đỏ của IDM nằm đúng trên đường cyan trong hình `scenarios_idm.png`, xác nhận biên `unavoidable` là đúng. Log người lái không phủ hết vùng khó: giới hạn phanh mạnh nhất trong log là `human_brake_limit_p99.9 = 3.6248 m/s²`, nhưng `8.56%` số ô lưới đòi phanh mạnh hơn mức đó (`share_beyond_human_brake_limit = 0.08556`). Cụ thể, số sự kiện khai thác được là `614` và chiếm `6.39%` các cửa sổ (`events=614`, `share=0.0639`), nên các điểm trắng trong lưới không phủ vùng “không có trong log”.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Bảng xếp hạng open-loop theo L2: `PercMLP` tốt nhất (`0.6239` m), sau đó `EgoMLP` (`0.7650`), `const-vel` (`1.2642`), `IDM` (`1.5386`). Bảng xếp hạng closed-loop theo tỷ lệ va chạm: `IDM` tốt nhất (`0.00%`), `PercMLP` (`0.82%`), `const-vel` (`20.05%`), `EgoMLP` (`22.43%`). Cặp đổi thứ tự rõ nhất là `const-vel` và `IDM`, `const-vel` và `EgoMLP`, rồi `IDM` so với `EgoMLP` và `PercMLP`; đây là cặp đảo hạng giữa open-loop và closed-loop. IDM có L2 tệ hơn các mô hình học nhưng vẫn an toàn hơn vì nó dùng phương trình dynamics và phanh dựa trên khoảng cách, không phải chỉ đoán tốc độ tương lai; nó giữ khoảng cách an toàn dù sai trong dự đoán 3 giây. EgoMLP tốt hơn `const-vel` về L2 vì nó thấy lịch sử tốc độ, nhưng vẫn đâm nhiều vì nó thiếu thông tin quan trọng về khoảng cách và tốc độ xe dẫn: khi xe dẫn cắt vào hoặc phanh mạnh, EgoMLP không có cảm biến tương đối để điều chỉnh kịp.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

`EgoMLP` có dải va chạm qua 3 seed là `0.1818` đến `0.2855` (tương ứng `18.18%` đến `28.55%`), tức khoảng cách giữa seed tốt nhất và xấu nhất là `10.37` điểm phần trăm. Wilson CI của seed tốt nhất là `[0.1681, 0.1963]`, còn của seed xấu nhất là `[0.2693, 0.3022]`; hai khoảng này không chồng nhau. Điều này nghĩa là nếu chỉ chạy một seed, ta có thể báo ra một kết luận rất khác: một seed cho thấy `18.2%` nhưng seed khác lên tới `28.5%`. Đó là nguy hiểm vì một mô hình học có thể trông “ổn” hoặc “đáng lo” tùy thuộc vào mômen huấn luyện, và trong hệ thống an toàn ta không được dùng một lần chạy làm căn cứ phát hành.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". T bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Tôi hiệu chỉnh `T = 1.4 s`, vì theo `calibration` trong `agents.json`, `wasserstein_by_T` thấp nhất ở `1.4s` (`0.35848`), thấp hơn nhiều so với phản ứng “không bao giờ phản ứng” (`W1 = 0.86510`). Trong thế giới “xe sau phát lại”, `IDM` có `rear=0.68296` và `any=0.75741` ở `t_react=0.6`; nhưng khi xe sau phản ứng, `rear=0.0` và `any=0.07444`. Đây là “va chạm giả” vì xe sau phát lại không biết xe trước vừa phanh và cứ giữ tốc độ cũ; nó chỉ va chạm bởi vì mô hình của thầy/đạo diễn không phản ứng, chứ không phải vì IDM thực sự gây ra sai lệch. Kết luận đổi ra là: nếu chỉ xem thế giới phát lại, ta sẽ đánh giá bộ điều khiển quá xấu; khi thay bằng một xe sau thật sự phản ứng, cường độ va chạm của IDM giảm xuống gần như bằng baseline và không còn là nguyên nhân gây rủi ro chính.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có tốt hơn IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị chặn bởi nhiều quy tắc: `R2_closed_loop_logs` với `candidate 1.1% (upper 1.6%)` so với baseline `0.0% + 1 point`; `R3_braking_slice` với `3.8%` và upper `7.7%` so với baseline `0.0% + 5 points`; `R4_scenario_grid` với `19.6%` vs baseline `0.0% + 2 points`; `R6_reactive_world` với `27.0%` vs baseline `7.4% + 2 points`. Hai nguyên nhân gốc là: (1) nó không gặp được các kịch bản cắt vào mà log không có, thể hiện ở `attributable_area = 0.19593` ở `t_react=0.6`, vượt quá vùng tránh được `0.07444` và “điểm cửa sổ không phủ vùng khó”; (2) nó có sai số tích luỹ khi tự lái, thể hiện qua mức va chạm trên log (`0.82%`, slice `3.8%`) và CI trên log không chạm baseline. Tôi sửa bằng cách bổ sung các cửa sổ từ giống kịch bản cắt vào mô phỏng và DAgger, tức là huấn luyện trên trạng thái mà mô hình tự đi vào, với nhãn từ dự báo 3s của IDM. Cách này giải quyết đúng hai nguyên nhân: nó mở rộng dữ liệu sang vùng chưa có trong log, và dạy mô hình phục hồi từ sai lầm của chính mình. PercMLP+ qua cổng, nhưng nó không “tốt hơn” IDM theo nghĩa tất yếu: `PercMLP+` có `closed-loop collision = 0.23%`, trong khi `IDM` là `0.00%`; nó chỉ chứng minh “không kém hơn baseline trong ngưỡng chấp nhận”, chứ chưa vượt thầy. Vì vậy, tôi không nên thay IDM ngay; cần thêm kiểm thử đa kịch bản, đo độ ổn định và kiểm tra với môi trường phản ứng hơn trước khi phát hành.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn ít nhất ba giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Giới hạn lớn nhất là mô phỏng này là hệ thống 1 chiều, không có cảm biến và không có chuyển hướng: trong thế giới thực, xe và người lái tương tác với các yếu tố bên cạnh, cho nên các đánh giá ở đây chỉ là đánh giá dọc trên một mô hình đơn giản. Thứ hai, xe dẫn trong log được phát lại không phản ứng với xe sau, nên các va chạm “phía sau” và các kịch bản tương tác bị bóp méo; `agents.json` cho thấy khi thay bằng xe sau phản ứng, kết quả `IDM` và `PercMLP` thay đổi rất mạnh, từ `rear=0.68` hoặc `0.54` xuống `0.0`. Thứ ba, lưới kịch bản và ngưỡng phanh là mô phỏng giả định, không phải dữ liệu thật; `human_brake_limit = 3.6248 m/s²` và `share_beyond_human_brake_limit = 8.56%` cho thấy log không bao phủ hết vùng khó, tức là biểu diễn không đại diện cho tất cả tình huống thực tế. Để thu hẹp khoảng cách với thế giới thật, cần thêm lượng dữ liệu với nhiều hành vi phản ứng, nhiều kịch bản cắt vào, có cảm biến/điều khiển ngang và kiểm thử trong môi trường tương tác có người lái thật và các xe khác phản ứng.

# REPORT — Lab Day 12

| Thông tin | Giá trị |
|---|---|
| Họ và tên | Phạm Văn Hoàng Anh Tú |
| MSSV (đúng bằng --student-id đã chạy) | 2A202602507 |
| GitHub | tubepvhat1604-bot |
| Seed chia dữ liệu / seed mô hình | 8554 / 1794, 1795, 1796, 1797, 1798 |
| Số seed huấn luyện đã dùng (--seeds) | 5 |

## Kết quả cuối

Lần chạy chính thức: `python run_all.py --student-id 2A202602507 --seeds 5`, không dùng `--quick`.
L2 là trung bình sai số vị trí ở 1, 2 và 3 giây; tỷ lệ va chạm tính trên 2883 quỹ đạo test.
Với mô hình học, bảng ghi min–max qua 5 seed; hai bộ điều khiển xác định có một kết quả.

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.277 | 20.916 | BLOCK |
| IDM | 1.567 | 0.035 | Baseline |
| EgoMLP | 0.771–0.778 | 11.863–41.970 | BLOCK |
| PercMLP | 0.630–0.636 | 0.520–3.295 | BLOCK |
| PercMLP+ (sau khi sửa) | 0.796–0.837 | 0.139–0.173 | RELEASE |

Nguồn: [openloop.json](evidence/openloop.json), [closedloop.json](evidence/closedloop.json), [gate.json](evidence/gate.json).
Dải L2 của PercMLP+ được tính lại từ 5 model đã lưu, dùng đúng tập test, tại [report_metrics.json](evidence/report_metrics.json); trung bình được đối chiếu với `openloop.json`.

## Câu 1 · Dữ liệu

`data_audit.json` ghi 992,679 dòng và 9,993 quỹ đạo, thời gian lấy mẫu 0.1 giây; sau khi loại 383 bản trùng còn 9,610 quỹ đạo.
Vấn đề thứ nhất là cột `Acc_LV` không đáng tin: tương quan với đạo hàm tốc độ chỉ 0.0454, trong khi phía xe theo sau đạt 0.9291.
Vì vậy tôi dùng gia tốc suy ra từ `Speed_LV` đã làm mượt (`logs.a_lead`) để khai thác ca phanh, tránh coi nhiễu một khung hình là phanh thật.
Vấn đề thứ hai là các quỹ đạo trùng có ID khác nhau: chia ngẫu nhiên theo ID mà chưa loại trùng có thể đưa cùng chuyển động vào cả train và test, khiến điểm đẹp giả tạo.
Vấn đề thứ ba là các khung hình liên tiếp trong một quỹ đạo phụ thuộc mạnh; chia theo khung làm rò rỉ các cửa sổ lịch sử/tương lai chồng nhau.
Do đó phải loại trùng trước rồi chia theo toàn bộ quỹ đạo; lần chạy này có 167,417 cửa sổ train và 71,716 cửa sổ test. Nguồn: `data_audit.json`, `openloop.json`.

## Câu 2 · Phát lại

Trong 600 đoạn, chạy lại cùng mã không lệch; refactor float32 khác giá trị chính xác ở 600 đoạn nhưng có 0 đoạn vượt dung sai, sai khác tuyệt đối lớn nhất 0.00009733 m/s².
Dung sai dùng là `atol=rtol=0.001`: một lệnh chỉ bị đánh dấu nếu vượt `atol + rtol * abs(golden)`. Sai số làm tròn và thứ tự phép toán không nhất thiết là đổi hành vi.
Thêm AEB làm lệch 12 đoạn. Tôi mở đoạn 57: các khung 98, 99, 100, 101 thay đổi; ở khung 98, gap=4.3033 m và tốc độ tiến gần=2.9189 m/s nên TTC=1.4743 s.
TTC dưới ngưỡng AEB 1.5 s khiến lệnh -5.4363 được ép về -6.0 m/s²; ở những khung TTC không thấp, nhánh này không tác động, và khi lệnh cũ đã phanh mạnh hơn thì cũng không đổi.
Chế độ fastest có 0 khung trễ và 0 khung rớt; realtime có 585 trễ và 583 rớt trên 60009 khung, làm 374 đoạn đổi.
Đây là chi phí xử lý được mô phỏng: fastest bỏ qua deadline nên che hiện tượng quá hạn và giữ lệnh cũ; không phải đo độ trễ phần cứng ngoài đời. Nguồn: `replay.json`, `replay_inspection.json`.

## Câu 3 · Kịch bản

Lưới có tốc độ cruise 12.0 m/s, khoảng cách 2.760–59.312 m và tốc độ tiến gần 0.0–12.0 m/s; 60×45=2700 ô.
Sau thời gian trễ, quãng đường co lại tối thiểu là `dv*t_react + dv²/(2*b_max)`; nếu gap nhỏ hơn biên này thì ngay cả phanh tối đa cũng không tránh được, theo giả định xe dẫn giữ tốc độ.
Tỷ lệ bất khả thi tăng từ 2.222% ở trễ 0 s lên 7.444% ở 0.6 s. Vùng va chạm của IDM tăng cùng mức, tức 5.222 điểm phần trăm; phần tránh được mà vẫn đâm của IDM bằng 0% trên cả hai lưới.
Kiểm chứng full-brake và biên vật lý có 0 ô lệch ngoài vùng sát biên tại cả hai độ trễ; vì thế không quy mọi ca đâm trong vùng bất khả thi thành lỗi bộ điều khiển.
Khai thác được 614 sự kiện phanh (6.389% quỹ đạo); mức phanh người lái ở phân vị 99.9 là 3.625 m/s², không phải cực đại tuyệt đối.
Có 8.556% lưới cần phanh mạnh hơn mức đó: các ca từ log chủ yếu đại diện vùng thường gặp, không bảo đảm phủ vùng đóng nhanh/khoảng cách ngắn. Nguồn: `scenarios.json`, `scenarios_idm.png`.

## Câu 4 · Vòng hở và vòng kín

Bảng dùng trung bình qua seed để xếp hạng, hạng 1 là tốt nhất; dải qua seed đã ghi ở bảng đầu.

| Bộ điều khiển | L2 trung bình (m) | Hạng vòng hở | Va chạm trung bình (%) | Hạng vòng kín |
|---|---|---|---|---|
| const-vel | 1.277 | 3 | 20.916 | 3 |
| IDM | 1.567 | 4 | 0.035 | 1 |
| EgoMLP | 0.774 | 2 | 24.211 | 4 |
| PercMLP | 0.632 | 1 | 1.602 | 2 |

Có 4 cặp đảo thứ tự: constant-velocity/IDM, constant-velocity/EgoMLP, IDM/EgoMLP và IDM/PercMLP (`closedloop.json`, mục `ranking`).
IDM có L2 1.567 m vì mục tiêu của luật là giữ khoảng cách, không sao chép chính xác người lái; nhưng chỉ đâm 1/2883 quỹ đạo (0.035%).
EgoMLP có L2 0.774 m, tốt hơn constant-velocity 1.277 m, nhưng không nhìn gap hay tốc độ xe dẫn nên không biết phản ứng đúng khi tình huống phía trước đổi.
Khi tự lái, sai số thay đổi trạng thái đầu vào tiếp theo rồi tích lũy, trong khi vòng hở liên tục được cung cấp lịch sử đúng từ log; đó là hai phép chấm khác nhau.
Lát cắt sắp phanh chỉ chiếm 4.369% cửa sổ; L2 PercMLP trên lát cắt là 0.868 m so với trung bình 0.632 m, cho thấy điểm trung bình che vùng khó. Nguồn: `openloop.json`, `closedloop.json`.

## Câu 5 · Một lần chạy chưa đủ

| Seed EgoMLP | Số va chạm / số quỹ đạo | Tỷ lệ (%) | Wilson 95% (%) |
|---|---|---|---|
| 1794 | 702/2883 | 24.350 | [22.818; 25.950] |
| 1795 | 709/2883 | 24.592 | [23.055; 26.198] |
| 1796 | 342/2883 | 11.863 | [10.733; 13.094] |
| 1797 | 527/2883 | 18.280 | [16.911; 19.732] |
| 1798 | 1210/2883 | 41.970 | [40.181; 43.781] |

EgoMLP dao động 11.863–41.970%, rộng 30.108 điểm phần trăm trên cùng tập test và cùng cách chia dữ liệu.
Seed tốt nhất 1796 có CI95 [10.733; 13.094]%, seed xấu nhất 1798 có CI95 [40.181; 43.781]%; hai khoảng không chồng nhau.
Wilson thể hiện bất định của tỷ lệ trên số quỹ đạo được chấm ở mỗi model; dải giữa seed thể hiện thêm bất ổn do huấn luyện, không thể thay thế nhau.
Nếu chỉ chọn seed 1796, có thể báo 11.863% và tạo cảm giác EgoMLP tốt hơn constant-velocity 20.916%; chọn seed 1798 lại cho 41.970% và kết luận ngược.
Vì thế cần báo cả dải, CI và seed xấu nhất; các khoảng này còn dựa trên giả định lấy mẫu, không bao phủ sai lệch do dữ liệu hay mô phỏng. Nguồn: `closedloop.json` → `logs.EgoMLP.per_seed`.

## Câu 6 · Tác nhân phản ứng

Bảng dưới dùng trễ phản ứng xe thử 0.6 s; với model học, script chọn seed có tỷ lệ ANY trong thế giới phản ứng cao nhất, rồi ghi cả hai thế giới của seed đó.

| Bộ điều khiển | ANY: xe sau phát lại (%) | ANY: xe sau phản ứng (%) | Đâm sau: phát lại (%) | Đâm sau: phản ứng (%) |
|---|---|---|---|---|
| const-vel | 77.630 | 77.630 | 0.000 | 0.000 |
| IDM | 75.741 | 7.444 | 68.296 | 0.000 |
| EgoMLP | 81.630 | 81.630 | 0.000 | 0.000 |
| PercMLP | 75.074 | 24.741 | 51.444 | 0.000 |
| PercMLP+ | 75.148 | 7.741 | 67.407 | 0.000 |

Hiệu chỉnh chọn T=1.4 s, có khoảng cách Wasserstein giữa phân bố headway bằng 0.343, thấp hơn xe không phản ứng 0.883; headway trung vị của người lái là 1.855 s.
Đây là khớp một đặc trưng thống kê, chưa chứng minh tác nhân giống người trên mọi hành vi.
Với IDM, đâm sau 68.296% khi xe sau phát lại giảm còn 0.000% khi nó phản ứng: xe phát lại không nhận biết xe thử đã phanh nên tiếp tục lao vào, tạo đánh giá giả về lỗi của xe thử.
ANY của IDM là 75.741% khi phát lại, khá gần constant-velocity 77.630%; khi phản ứng, IDM chỉ còn 7.444% còn constant-velocity vẫn 77.630%.
Trong lần chạy này IDM vẫn xếp tốt hơn constant-velocity ở cả hai thế giới, nhưng khoảng cách và kết luận về mức nguy hiểm thay đổi rất lớn; không nên khẳng định có đảo hạng nếu bảng không cho thấy. Nguồn: `agents.json`.

## Câu 7 · Cổng phát hành và sửa

| Quy tắc | PercMLP trước sửa | PercMLP+ sau sửa |
|---|---|---|
| R1 | Qua | Qua |
| R2 | Rớt | Qua |
| R3 | Rớt | Qua |
| R4 | Rớt | Qua |
| R5 | Rớt | Qua |
| R6 | Rớt | Qua |
| R7 | Qua | Qua |

PercMLP bị chặn bởi R2, R3, R4, R5, R6: R2 có cận trên va chạm 4.011% vượt baseline 0.035% + 1 điểm; R3 có cận trên lát cắt 20.089% vượt giới hạn 5%.
R4 có vùng tránh được mà vẫn đâm 17.296% vượt 2%; R5 có dải 0.520–3.295%, rộng hơn 1 điểm; R6 có ANY phản ứng 24.741% so với IDM 7.444% + 2 điểm.
Hai nguyên nhân là thiếu bao phủ các trạng thái cắt vào khó (R4/R6), và sai số bắt chước tích lũy khi model tự tạo trạng thái tiếp theo (R2/R3); R5 bộc lộ thêm độ bất ổn giữa các seed.
Cách sửa thêm 20,000 kịch bản cắt vào gắn nhãn IDM và 2 vòng DAgger mỗi seed: cho model tự lái trên train, thu trạng thái nó đi vào rồi hỏi IDM hành động tương lai ở chính trạng thái đó, không dùng test để huấn luyện.
Sau sửa, cổng quyết định RELEASE; va chạm trên log 0.139–0.173%, cận trên seed xấu nhất 0.405%; vùng tránh được mà vẫn đâm còn 0.296%.
L2 vòng hở trung bình đổi từ 0.632 thành 0.815 m: điểm bắt chước log kém hơn vẫn có thể đi kèm điều khiển an toàn hơn trong bộ kiểm tra.
PercMLP+ học từ IDM và qua cổng chỉ chứng minh đạt các ngưỡng không kém hơn trong phạm vi đã xét, chưa chứng minh tốt hơn thầy hay bảo đảm ngoài miền; ở 20 m/s, vùng lỗi là 0.037–3.111% so với IDM 0.000–0.000%.
Vì vậy tôi chưa đề xuất thay IDM để phát hành thật: cần thêm bộ test độc lập, tốc độ ngoài miền, tác nhân đa dạng và thử nghiệm có giám sát trước quyết định. Nguồn: `gate.json`, `fix.json`, `openloop.json`, `closedloop.json`.

## Câu 8 · Bài lab này không chứng minh được gì

Thứ nhất, mô phỏng chỉ xét chuyển động dọc và quan sát trạng thái lý tưởng; nhịp 0.1 s và trễ 0/0.6 s không bao quát lỗi cảm biến, mất nhận dạng, động lực học ngang hay mọi độ trễ thực. Cần kiểm tra có cảm biến và nhiều loại tình huống hơn.
Thứ hai, xe dẫn trên log không phản ứng với xe thử; chỉ thay xe sau bằng IDM với T=1.4 s vẫn chưa đại diện tương tác của nhiều kiểu người lái. Cần tác nhân đa dạng và kiểm tra độ nhạy đối với các tham số hành vi.
Thứ ba, log có 9,610 quỹ đạo ngắn và chỉ khai thác được 614 ca phanh; lưới 2700 ô không phải phân bố giao thông thực. Phần trăm diện tích lưới không phải xác suất tai nạn ngoài đời, nên cần dữ liệu độc lập theo tuyến đường, thời tiết và nhóm tình huống.
Thứ tư, model sửa học kịch bản đến 16 m/s nhưng kiểm tra ngoài miền tại 20 m/s cho vùng lỗi tránh được 0.037–3.111%; điều này cho thấy qua cổng ở 12 m/s không bảo đảm tổng quát hóa. Cần mở rộng miền và khóa tập đánh giá mới trước khi tiếp tục điều chỉnh.
Cuối cùng, 5 seed và CI95 chỉ mô tả một phần bất định trên dữ liệu/mô phỏng đã chọn; các ngưỡng 1 và 2 điểm của cổng là lựa chọn của đội, không phải chứng nhận an toàn. Tôi đề xuất đánh giá độc lập, thử nghiệm kín có giám sát và tiêu chí dừng rõ ràng trước khi xét triển khai thực tế.
Nguồn cho các điều kiện thí nghiệm: `lab12/config.py`, `agents.json`, `scenarios.json`, `fix.json`, `gate.json`.

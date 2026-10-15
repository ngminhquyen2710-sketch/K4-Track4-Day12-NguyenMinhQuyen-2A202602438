# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Nguyễn Minh Quyền |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602438 |
| GitHub | ngminhquyen2710-sketch |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 1670 / 4209 |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.258 | 20.4 % | BLOCK |
| IDM | 1.520 | 0.1 % | (baseline) |
| EgoMLP | 0.764 .. 0.768 | 18.0 .. 27.8 % | BLOCK |
| PercMLP | 0.616 .. 0.622 | 0.9 .. 1.9 % | BLOCK |
| PercMLP+ (sau khi sửa) | 0.814 .. 0.822 | 0.2 .. 0.3 % | RELEASE |

## Câu 1 · Dữ liệu
Bước 0 đã phát hiện ba vấn đề cốt lõi trong tập dữ liệu Argoverse:
1. Cột gia tốc xe dẫn `Acc_LV` có hệ số tương quan tuyến tính với đạo hàm vận tốc d(Speed_LV)/dt chỉ đạt xấp xỉ 0.0454 (gần như bằng 0, hoàn toàn không tương thích vật lý), trong khi xe sau `Acc_FAV` đạt độ tương quan 0.9291. Do đó, `Acc_LV` không thể sử dụng trực tiếp; ta phải thay thế bằng gia tốc tính từ đạo hàm của vận tốc xe dẫn đã qua bộ lọc làm mượt (smoothed `Speed_LV`).
2. Phát hiện chính xác 383 quỹ đạo trùng lặp tuyệt đối (exact duplicates) nằm rải rác ở các ID cách xa nhau (từ 9.993 quỹ đạo thô ban đầu lọc còn 9.610 quỹ đạo sạch). Nếu không loại bỏ, các quỹ đạo trùng lặp này có thể đồng thời xuất hiện ở cả tập train và test gây rò rỉ dữ liệu (data leakage) nghiêm trọng.
3. Việc phân chia train/test bắt buộc phải thực hiện theo từng quỹ đạo (trajectory) thay vì chia ngẫu nhiên theo khung hình (frame). Trong cùng một quỹ đạo 10 Hz (mỗi bước 0.1 s), các khung hình liên tiếp có tính tự tương quan thời gian cực lớn; chia theo khung hình sẽ khiến mô hình ghi nhớ trạng thái lân cận thay vì học được khả năng tổng quát hóa thực sự.

## Câu 2 · Phát lại
Bản refactor float32 cho kết quả 600/600 đoạn khác biệt từng bit (`not_bit_identical = 600`), nhưng có 0/600 đoạn vượt dung sai (`differ_beyond_tol = 0`) với độ lệch tuyệt đối cực đại chỉ 4.03e-4 m/s². Bản refactor vẫn hoàn toàn đúng vì phép cộng số thực dấu phẩy động không có tính kết hợp (ví dụ: `(0.1 + 0.2) + 0.3 = 0.6000000000000001` khác `0.1 + (0.2 + 0.3) = 0.6`). Việc thay đổi kiểu dữ liệu float32 và thứ tự tính toán làm lệch bit nhưng không làm thay đổi hành vi điều khiển; mức dung sai `atol = 1e-3, rtol = 1e-3` là hoàn toàn hợp lý.
Thay đổi AEB (`aeb_ttc = 1.5s`) chỉ làm lệch 10/600 đoạn (`[45, 56, 59, 151, 171, 235, 285, 302, 413, 461]`). Mở đoạn 45 cho thấy sai lệch xuất hiện chính xác tại các khung hình 98 đến 101: khi khoảng cách co hẹp từ 4.30 m xuống 3.62 m với tốc độ tiến gần 2.92 m/s đến 2.43 m/s, giá trị TTC giảm xuống còn 1.43 s đến 1.49 s (< 1.5 s), kích hoạt phanh khẩn cấp `cmd = -6.00 m/s²` (thay vì IDM thông thường chỉ phanh -5.44 đến -5.91 m/s²). Các khung hình có TTC >= 1.5 s hoàn toàn không bị ảnh hưởng.
Chế độ phát lại nhanh nhất (`fastest`) bỏ qua hoàn toàn khái niệm thời gian thực, dẫn đến 0 khung trễ và 0 khung bị rớt (`late = 0, dropped = 0`). Ngược lại, chế độ thời gian thực (`realtime`) phơi bày đúng độ trễ ngẫu nhiên của hệ thống, làm lộ ra 582 khung trễ hạn và 577 khung bị rớt trên 59.580 khung hình, khiến lệnh điều khiển bị thay đổi ở 373/600 phân đoạn log.

## Câu 3 · Kịch bản
Lưới kịch bản có vận tốc cruise 14.0 m/s, dải khoảng cách từ 2.76 m đến 59.31 m (60 mốc), dải tốc độ tiến gần từ 0.0 m/s đến 14.0 m/s (45 mốc), tạo thành tổng cộng 2.700 ô kiểm thử.
Trong đó, vùng va chạm vật lý không thể tránh khỏi (`unavoidable`) khi không có độ trễ ($t_{\text{react}} = 0.0\text{ s}$) chiếm 3.63% diện tích lưới (98/2.700 ô). Khi xét độ trễ phản ứng thực tế $t_{\text{react}} = 0.6\text{ s}$, vùng bất khả thi này mở rộng lên 10.22% (276/2.700 ô), tức tăng thêm 6.59% diện tích. Ranh giới vật lý này là chìa khóa để phân định: nếu xe đâm trong vùng này thì không bộ điều khiển nào có thể tránh được do giới hạn động học, còn va chạm ngoài vùng này mới chính là lỗi thuộc về bộ điều khiển.
Dữ liệu log thực tế không bao phủ được các tình huống phanh tới hạn nguy hiểm: mức phanh gắt nhất mà người lái trong log từng thực hiện (phân vị 99.9%) chỉ đạt 3.62 m/s², trong khi cơ cấu chấp hành xe có thể phanh tới 9.0 m/s². Có tới 12.67% diện tích lưới kịch bản đòi hỏi mức phanh lớn hơn 3.62 m/s². Toàn bộ 614 sự kiện phanh khai thác từ log (chiếm 6.39% số quỹ đạo) chỉ phân bố ở vùng tốc độ tiến gần thấp và khoảng cách an toàn, hoàn toàn bỏ trống vùng góc nguy hiểm (khoảng cách ngắn kết hợp chênh lệch tốc độ cao).

## Câu 4 · Vòng hở và vòng kín
Bảng tổng hợp điểm số và thứ hạng của 4 bộ điều khiển:
- `PercMLP`: L2 vòng hở = 0.619 m (Hạng 1) | Va chạm vòng kín = 1.3 % (Hạng 2)
- `EgoMLP`: L2 vòng hở = 0.766 m (Hạng 2) | Va chạm vòng kín = 23.5 % (Hạng 4)
- `const-vel`: L2 vòng hở = 1.258 m (Hạng 3) | Va chạm vòng kín = 20.4 % (Hạng 3)
- `IDM`: L2 vòng hở = 1.520 m (Hạng 4) | Va chạm vòng kín = 0.1 % (Hạng 1)

Có tới 4 cặp bị đảo ngược thứ hạng hoàn toàn giữa hai phương pháp chấm điểm: `('const-vel', 'IDM')`, `('const-vel', 'EgoMLP')`, `('IDM', 'EgoMLP')`, và `('IDM', 'PercMLP')`.
Cơ chế đằng sau sự nghịch lý này:
- IDM bị điểm L2 vòng hở đội sổ (1.520 m) vì nó là một phương trình vi phân điều khiển phản hồi trạng thái, không tối ưu hóa hàm mất mát để bắt chước từng khung hình cụ thể của con người trong điều kiện lái êm đềm bình thường. Tuy nhiên, khi đưa vào vòng kín, IDM liên tục quan sát khoảng cách và vận tốc tương đối tại mỗi bước thời gian để điều chỉnh gia tốc, nên giữ an toàn xuất sắc với tỷ lệ va chạm chỉ 0.1% (và trên lát cắt phanh gấp là 0.0%).
- Ngược lại, EgoMLP đạt L2 vòng hở rất đẹp (0.766 m) vì mạng nơ-ron học tốt việc dự báo quán tính tốc độ trung bình của xe từ lịch sử. Nhưng do EgoMLP hoàn toàn không quan sát khoảng cách (`gap`) và tốc độ xe trước (`lead_v`), nó không hề biết xe phía trước đang ở đâu. Khi xe trước phanh gấp trong vòng kín, EgoMLP vẫn tiếp tục lao tới và gây va chạm tới 23.5% (trên lát cắt phanh gấp va chạm lên đến 59.1%, còn tệ hơn cả const-vel 55.1%).

## Câu 5 · Một lần chạy chưa đủ
Khi huấn luyện EgoMLP trên 5 seed ngẫu nhiên khác nhau, tỷ lệ va chạm vòng kín dao động rất mạnh trong dải từ 18.0% đến 27.8% (chênh lệch gần 10% tuyệt đối). Tương tự, PercMLP dao động từ 0.9% đến 1.9%.
Khoảng tin cậy Wilson 95% của EgoMLP ở seed xấu nhất (seed 4212, va chạm 27.8%) là [26.2, 29.4]%, trong khi seed tốt nhất (seed 4213, va chạm 18.0%) có khoảng tin cậy là [16.6, 19.4]%. Hai khoảng tin cậy này hoàn toàn tách biệt và không hề giao nhau.
Nếu kỹ sư chỉ chạy duy nhất một seed, kết luận rút ra có thể bị sai lệch nghiêm trọng: nếu rơi vào seed 18.0%, người kiểm thử có thể đánh giá thấp mức độ nguy hiểm; nếu rơi vào seed 27.8%, kết quả lại quá bi quan. Do mạng nơ-ron chịu ảnh hưởng lớn từ khởi tạo trọng số ngẫu nhiên và thứ tự lấy mẫu mini-batch, việc chỉ báo cáo một con số từ một lần chạy duy nhất mang tính ngẫu nhiên cao, che giấu sự mất ổn định cố hữu và cực kỳ nguy hiểm trong các ứng dụng an toàn xe tự hành.

## Câu 6 · Tác nhân phản ứng
Bảng so sánh diện tích va chạm trên lưới kịch bản cắt vào ($t_{\text{react}} = 0.6\text{ s}$):
- `const-vel`: Xe sau phát lại = 80.6 % | Xe sau phản ứng = 80.6 % | Va chạm đuôi: 0.0 % / 0.0 %
- `IDM`: Xe sau phát lại = 80.9 % | Xe sau phản ứng = 10.2 % | Va chạm đuôi: 70.6 % / 0.0 %
- `EgoMLP`: Xe sau phát lại = 79.2 % | Xe sau phản ứng = 79.2 % | Va chạm đuôi: 0.0 % / 0.0 %
- `PercMLP`: Xe sau phát lại = 74.4 % | Xe sau phản ứng = 31.9 % | Va chạm đuôi: 43.0 % / 0.0 %

Thời khoảng cách trung vị của người lái ghi nhận trong log là 1.86 s. Thuật toán đã quét dải tham số thời khoảng cách $T$ từ 0.6 s đến 2.4 s và hiệu chỉnh được giá trị tối ưu $T = 1.4\text{ s}$ với khoảng cách Wasserstein W1 nhỏ nhất bằng 0.33 (trong khi xe không phản ứng đạt W1 = 0.88), phản ánh chân thực hành vi bám xe của con người.
Khi xe phía sau chỉ đơn thuần "phát lại" hành trình cũ, nó hoàn toàn mù quáng trước việc xe thử nghiệm phía trước đang phanh khẩn cấp để tránh va chạm cắt đầu. Hậu quả là xe sau đâm sầm vào đuôi xe thử nghiệm, tạo ra tới 70.6% "va chạm giả từ phía sau" đối với IDM, khiến IDM trông như một bộ điều khiển tồi tệ (va chạm 80.9%). Khi chuyển sang môi trường tác nhân có phản ứng, xe sau biết tự phanh giữ khoảng cách an toàn, xóa bỏ hoàn toàn 100% va chạm đuôi (về 0.0%), đưa tỷ lệ va chạm thực sự của IDM về đúng 10.2% (trùng khớp với giới hạn vật lý bất khả thi). Điều này chứng minh rằng kiểm thử trong thế giới không phản ứng sẽ dẫn đến kết luận hoàn toàn sai về năng lực của hệ thống tự hành.

## Câu 7 · Cổng phát hành và sửa
Ứng viên PercMLP bị chặn (BLOCK) bởi 4 quy tắc: R2 (va chạm trên log 1.9%, cận trên 2.4% vượt ngưỡng baseline 0.1% + 1.0%), R3 (va chạm lát cắt phanh 9.1%, cận trên 14.3% vượt ngưỡng baseline 0.0% + 5.0%), R4 (va chạm lưới cắt vào có thể tránh được là 21.6% vượt ngưỡng baseline 0.0% + 2.0%) và R6 (va chạm với xe sau phản ứng là 31.9% vượt ngưỡng baseline 10.2% + 2.0%).
Có hai nguyên nhân gốc rễ:
1. Thiếu hụt kịch bản cắt vào trong tập dữ liệu log (Out-of-Distribution): log Argoverse hầu như chỉ gồm các tình huống bám xe thẳng thông thường, không chứa tình huống xe khác tạt đầu ở cự ly hẹp.
2. Lỗi trôi dạt tích lũy trong vòng kín (Covariate Shift): khi tự lái, các sai số dự báo nhỏ tích tụ dần khiến xe rơi vào những trạng thái lạ chưa từng có trong tập huấn luyện và không biết cách tự đưa xe trở lại trạng thái an toàn.

Ta đã sửa chữa mô hình bằng hai kỹ thuật bổ sung dữ liệu:
1. Tạo 196.860 cửa sổ cắt vào tổng hợp từ 20.000 kịch bản mô phỏng (`cutin_windows`), gán nhãn bằng hành động xử lý an toàn của chuyên gia IDM để bổ sung kiến thức về tình huống tạt đầu.
2. Áp dụng thuật toán DAgger qua 2 vòng (`dagger_windows`): cho mô hình tự lái trên tập log, ghi nhận 894.700+ cửa sổ tại chính các trạng thái mà mô hình tự đi vào, và nhờ chuyên gia IDM gán nhãn dự báo 3 giây tiếp theo tại các trạng thái đó, dạy mô hình cách tự phục hồi từ sai số của chính mình.
Sau khi sửa, PercMLP+ đã vượt qua toàn bộ 7 quy tắc và được PHÁT HÀNH (RELEASE): tỷ lệ va chạm trên log giảm còn 0.2 .. 0.3% (cận trên 0.5% <= baseline 0.1% + 1%), va chạm lưới cắt vào giảm ngoạn mục từ 21.6% xuống còn 0.2%. Đáng chú ý, điểm L2 vòng hở của PercMLP+ tăng từ 0.619 m lên 0.817 m (điểm L2 kém đi nhưng mô hình lái an toàn hơn gấp nhiều lần, khẳng định thêm việc điểm vòng hở không đo lường được an toàn vòng kín).
PercMLP+ có tốt hơn IDM không? **KHÔNG**, kết quả chỉ chứng minh PercMLP+ "không kém hơn" IDM (non-inferior), bởi vì toàn bộ nhãn bổ sung của PercMLP+ đều được sinh ra từ việc học theo luật IDM, nên nó không thể vượt qua năng lực của "người thầy" IDM. Chưa nên phát hành thay thế IDM ngay trên xe thương mại; cần tiến hành kiểm thử mở rộng với nhiều loại cảm biến, các vận tốc cruise khác nhau (ví dụ 20 m/s), và thử nghiệm thực địa trên đường thử khép kín trước khi đưa ra quyết định cuối cùng.

## Câu 8 · Bài lab này không chứng minh được gì
Ba giới hạn quan trọng nhất của bài lab đối với quyết định phát hành hệ thống ra thực tế:
1. Giả định chuyển động một chiều dọc thuần túy (1D Longitudinal Motion Only): Bài lab giả định các xe chỉ di chuyển trên một làn duy nhất dọc theo trục đường, hoàn toàn bỏ qua động lực học chuyển động ngang (đánh lái chuyển làn, trượt ngang, giới hạn bám đường của lốp xe khi vào cua). Trong giao thông thực tế, nhiều tình huống va chạm phía trước nguy cấp có thể được giải quyết an toàn bằng thao tác đánh lái né tránh sang làn phụ thay vì chỉ phanh gấp dọc.
2. Giả định cảm biến hoàn hảo không độ trễ và không nhiễu (Perfect Perception / No Occlusion): Bài lab cung cấp cho bộ điều khiển giá trị khoảng cách và tốc độ tương đối chính xác tuyệt đối từ phép đo hình học. Trên xe thật, dữ liệu từ Camera, Radar, LiDAR luôn có nhiễu đo đạc, độ trễ xử lý thuật toán nhận diện, hiện tượng nhận diện nhầm (false positives/false negatives), hiện tượng mất dấu mục tiêu (track loss) và các điểm mù do xe khác che khuất.
3. Xe dẫn trong log không phản ứng với xe thử nghiệm (Non-reactive Lead Vehicle): Xe dẫn trong dữ liệu log chỉ phát lại tốc độ thụ động theo kịch bản có sẵn trong quá khứ mà không có bất kỳ tương tác ngược nào với hành vi của xe tự lái phía sau. Trong đời thực, hành vi phanh hoặc áp sát của xe sau sẽ tác động trực tiếp đến tâm lý và phản xạ của tài xế xe dẫn.

Để thu hẹp khoảng cách tới thế giới thực, các bước kỹ thuật cần thực hiện tiếp theo bao gồm:
- Tích hợp mô phỏng 3D đa hướng với mô hình động lực học xe hoàn chỉnh (vật lý hệ thống treo, lốp xe Pacejka, điều khiển phanh và góc lái tích hợp).
- Bổ sung mô hình mô phỏng cảm biến vật lý có trễ hạn, độ phân giải hữu hạn, nhiễu đo và hiện tượng che khuất thời gian thực.
- Sử dụng mô hình mô phỏng giao thông đa tác nhân tương tác (multi-agent interactive simulation) dựa trên lý thuyết trò chơi hoặc học tăng cường (RL) để các xe cùng phản ứng qua lại lẫn nhau.
- Triển khai thử nghiệm nghiêm ngặt trên phần cứng thực tế (Hardware-in-the-Loop - HIL) và chạy thử nghiệm an toàn trên đường thử khép kín (proving ground) trước khi xem xét phát hành trên đường công cộng.

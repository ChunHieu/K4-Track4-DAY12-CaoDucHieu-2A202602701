# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | `Cao Đức Hiếu` |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `2A202602701` |
| GitHub | `ChunHieu` |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | `1918 / 9543` |
| Số seed huấn luyện bạn dùng (`--seeds`) | `3` (5 seeds [9543..9547] cho bước 3, 4, 6; 3 seeds [9543, 9544, 9545] cho bước 7 fix) |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.267 | 19.8% | BLOCK |
| IDM | 1.528 | 0.0% | (baseline) |
| EgoMLP | 0.762 .. 0.770 (mean 0.765) | 20.3 .. 32.2% (mean 24.4%) | BLOCK |
| PercMLP | 0.614 .. 0.623 (mean 0.619) | 0.6 .. 1.1% (mean 0.9%) | BLOCK |
| PercMLP+ (sau khi sửa) | 0.801 .. 0.827 (mean 0.811) | 0.3 .. 0.3% (mean 0.3%) | RELEASE |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Trong quá trình kiểm toán 992,679 dòng dữ liệu thô từ 9,993 quỹ đạo OpenPAV (Argoverse 2), tôi phát hiện ba vấn đề nghiêm trọng: dữ liệu gia tốc xe dẫn bị nhiễu sai lệch hoàn toàn, xuất hiện nhiều quỹ đạo nhân bản ở các ID cách xa nhau, và phân phối hành vi bị lệch về các tình huống lái êm đềm. Cụ thể, cột gia tốc thô `Acc_LV` có hệ số tương quan với đạo hàm vận tốc thật (`d(speed)/dt`) chỉ đạt 0.04535 (gần như bằng 0, trong khi xe theo sau đạt tới 0.92909), nghĩa là giá trị cảm biến gia tốc xe dẫn ghi nhận hoàn toàn không đáng tin cậy. Do đó, hệ thống buộc phải loại bỏ `Acc_LV` và thay thế bằng gia tốc làm mượt `a_lead` được tính toán thông qua đạo hàm của mảng vận tốc lọc nhiễu `v_lead_s`. Có đúng 383 quỹ đạo trùng lặp tuyệt đối đã bị phát hiện và loại bỏ, giúp tập dữ liệu sạch còn lại 9,610 quỹ đạo. Việc phân chia tập huấn luyện và kiểm thử bắt buộc phải thực hiện theo đơn vị từng quỹ đạo (trajectory-based split) thay vì chia ngẫu nhiên theo khung hình (frame-based split) nhằm triệt tiêu hoàn toàn hiện tượng rò rỉ dữ liệu (data leakage) do tính tương quan chuỗi thời gian giữa các bước 0.1s liền kề.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Khi phát lại 600 đoạn nhật ký (tổng dung lượng 9.5 MB), bản refactor tối ưu kiểu số thực float32 bị sai lệch từng bit ở toàn bộ 600/600 đoạn (chiếm 100%), tuy nhiên số đoạn vượt ngưỡng dung sai (atol=1e-3, rtol=1e-3) lại bằng đúng 0/600 đoạn, với sai số tuyệt đối cực đại chỉ là 0.00012 m/s² (1.2e-4). Hiện tượng này phản ánh đúng bản chất toán học của chuẩn số thực IEEE-754 khi phép cộng không có tính kết hợp: ví dụ thực nghiệm cho thấy (0.1 + 0.2) + 0.3 = 0.6000000000000001 trong khi 0.1 + (0.2 + 0.3) = 0.6. Khi bổ sung tính năng phanh khẩn cấp tự động AEB với ngưỡng TTC 1.5s, kết quả chỉ làm lệch đúng 3/600 đoạn (cụ thể là các đoạn số 150, 206 và 318). Mở chi tiết các đoạn này cho thấy lệnh phanh cực đại -9.0 m/s² chỉ kích hoạt cục bộ tại đúng các khung hình có khoảng cách gap co hẹp nhanh khiến TTC rơi xuống dưới 1.5s, các khung hình còn lại hoàn toàn giữ nguyên lệnh điều khiển IDM. Cuối cùng, chế độ phát lại nhanh nhất ("fastest") xử lý 59,455 khung hình mà không xuất hiện bất kỳ khung hình trễ hay rớt nào (0 late, 0 dropped), che giấu hoàn toàn áp lực tính toán thực tế. Ngược lại, chế độ thời gian thực ("real-time") bộc lộ tới 581 khung hình bị trễ hạn và 574 khung hình bị hủy bỏ, dẫn tới 378/600 đoạn bị biến đổi hành vi lái do trễ chu kỳ điều khiển 0.1s.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới kịch bản cắt đầu được thiết lập ở vận tốc hành trình 10.0 m/s với 60 mức khoảng cách gap (từ 2.72 m đến 56.40 m) và 45 mức tốc độ tiến gần closing speed (từ 0.0 đến 10.0 m/s), tạo thành không gian kiểm thử gồm 2,700 ô (cells). Tại độ trễ phản xạ t_react = 0.0 s, vùng vật lý bất khả kháng (unavoidable frontier) chiếm đúng 1.0% tổng số ô (tương ứng 28 ô); khi độ trễ tăng lên t_react = 0.6 s, vùng va chạm tất yếu này tăng vọt lên 5.1% số ô (138 ô). Đáng chú ý, tỷ lệ va chạm của thuật toán IDM khớp hoàn hảo 100% với giới hạn vật lý này (tăng từ 1.0% lên 5.1%, với đúng 0 ô sai lệch ngoài biên ở cả hai mức độ trễ), khẳng định tính đúng đắn của công thức biên động học. Tập log tự nhiên bộc lộ sự thiếu hụt nghiêm trọng các kịch bản nguy hiểm: gia tốc phanh mạnh nhất của tài xế con người trong log (phân vị 99.9%) chỉ đạt 3.6 m/s² (`human_brake_limit`), trong khi cơ cấu chấp hành xe có thể đạt tới 9.0 m/s². Có tới 5% số ô trên lưới kiểm thử đòi hỏi mức giảm tốc vượt quá 3.6 m/s². Trong 9,610 quỹ đạo, hệ thống chỉ khai phá được 614 sự kiện phanh gấp (tỷ lệ 6.4%), với mức phanh trung vị p50 là -2.5 m/s² và p99 là -4.2 m/s², cho thấy dữ liệu nhật ký giao thông hàng ngày hoàn toàn không bao phủ được các tình huống phanh tới hạn.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Bảng so sánh đối đầu giữa đánh giá vòng hở (Open-loop L2) và mô phỏng vòng kín (Closed-loop Simulation):
- const-vel: L2 vòng hở = 1.267 m (Hạng 3) | Va chạm vòng kín = 19.8% (Hạng 3)
- IDM: L2 vòng hở = 1.528 m (Hạng 4 - tệ nhất) | Va chạm vòng kín = 0.0% (0.03%, Hạng 1 - an toàn nhất)
- EgoMLP: L2 vòng hở = 0.765 m (Hạng 2) | Va chạm vòng kín = 24.4% (Hạng 4 - đâm nhiều nhất)
- PercMLP: L2 vòng hở = 0.619 m (Hạng 1 - tốt nhất) | Va chạm vòng kín = 0.9% (Hạng 2)

Thực nghiệm chỉ ra sự đảo ngược thứ hạng (ranking inversion) ở 4 cặp bộ điều khiển: ('const-vel', 'IDM'), ('const-vel', 'EgoMLP'), ('IDM', 'EgoMLP') và ('IDM', 'PercMLP'). IDM có sai số L2 vòng hở lớn nhất (1.528 m) vì nó là luật điều khiển vật lý độc lập, không tối ưu hóa hàm mất mát để bắt chước từng bước ga phanh của tài xế người trong log. Tuy nhiên, trong vòng kín, IDM sở hữu vòng phản hồi kín (closed-loop feedback) cực kỳ chuẩn xác, lập tức đạp phanh khi khoảng cách co hẹp nên giữ tỷ lệ va chạm ở mức 0.0%. Ngược lại, EgoMLP có L2 vòng hở khá thấp (0.765 m) do mạng nơ-ron học khớp tốt phân phối tốc độ trung bình của con người. Thế nhưng, EgoMLP hoàn toàn mù thông tin phía trước (không nhận đầu vào gap và v_lead). Khi đưa vào vòng kín, các sai số tích luỹ qua từng bước 0.1s đẩy xe trôi dạt (drift), và khi xe dẫn phanh gấp, EgoMLP không hề nhận biết nên tiếp tục duy trì tốc độ, dẫn tới tỷ lệ va chạm lên tới 24.4% (và lên tới 50.3% trên lát cắt phanh gấp).

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Mô hình học máy EgoMLP bộc lộ sự dao động rất lớn qua 5 hạt giống huấn luyện: tỷ lệ va chạm biến thiên từ 20.3% (seed 9547, k=585/2883) lên tới 32.2% (seed 9543, k=928/2883), tạo ra khoảng chênh lệch hiệu năng lên tới 11.9 điểm phần trăm giữa các lần khởi tạo trọng số ngẫu nhiên. Khi áp dụng khoảng tin cậy Wilson 95%, seed tốt nhất (9547) có cận tin cậy [18.9%, 21.8%], trong khi seed xấu nhất (9543) có cận tin cậy [30.5%, 33.9%]. Hai khoảng tin cậy này hoàn toàn tách rời nhau (không hề chồng lấn), chứng minh sự chênh lệch này là do tính ngẫu nhiên của mô hình chứ không phải do sai số lấy mẫu ngẫu nhiên. Ngay cả mô hình PercMLP cũng dao động từ 0.6% (CI95 [0.4%, 0.9%] ở seed 9544) đến 1.1% (CI95 [0.8%, 1.6%] ở seed 9547). Nếu nhóm phát triển chỉ thực hiện một lần chạy duy nhất và may mắn bốc trúng seed 9547 của EgoMLP để báo cáo kết quả 20.3%, họ sẽ tạo ra ảo tưởng rằng mô hình tương đương với const-vel (19.8%). Điều này đặc biệt nguy hại vì khi triển khai thực tế, mô hình có thể gặp biến thể trọng số xấu dẫn tới tỷ lệ va chạm tăng vọt lên 32.2%, gây rủi ro an toàn sinh mạng nghiêm trọng.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Bảng so sánh tỷ lệ va chạm tổng thể (trước/sau) trên lưới cắt đầu tại t_react = 0.6 s:
- const-vel: xe sau phát lại 73.4% (đâm sau 0.0%) | xe sau phản ứng 73.4% (đâm sau 0.0%)
- IDM: xe sau phát lại 70.1% (đâm sau 64.9%) | xe sau phản ứng 5.1% (đâm sau 0.0%)
- EgoMLP: xe sau phát lại 71.9% (đâm sau 0.0%) | xe sau phản ứng 71.9% (đâm sau 0.0%)
- PercMLP: xe sau phát lại 74.2% (đâm sau 53.5%) | xe sau phản ứng 21.2% (đâm sau 0.0%)

Thông qua hàm khoảng cách phân phối Wasserstein-1 (W1), tham số thời khoảng cách an toàn của xe sau được hiệu chỉnh tối ưu ở giá trị T = 1.4 s (khoảng cách W1 đạt 0.36, sát với phân phối thực tế của con người có trung vị 1.86 s, vượt trội hơn hẳn mức 0.86 của một xe không phản ứng). Trong thế giới xe sau chỉ phát lại (replayed), khi xe thử nghiệm chạy IDM phanh gấp để tránh xe dẫn, chiếc xe phía sau vẫn mù quáng duy trì quỹ đạo trong log và đâm sầm vào đuôi xe IDM, tạo ra tới 64.9% va chạm giả phía sau (rear collisions) và 53.5% cho PercMLP. Ngược lại, const-vel và EgoMLP do không phanh nên xe sau không đâm đuôi (0.0%), tạo ra kết luận sai lệch nghiêm trọng rằng IDM (70.1%) nguy hiểm ngang với const-vel (73.4%). Khi chuyển sang môi trường tác nhân phản ứng (reactive agent), 100% va chạm phía sau của IDM biến mất, kéo tỷ lệ va chạm thực chất từ 70.1% rơi thẳng xuống chỉ còn 5.1%, khôi phục sự thật về năng lực điều khiển an toàn vượt bậc của IDM.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

Mô hình PercMLP bị chặn (BLOCK) bởi 4 quy tắc an toàn: R2 (closed-loop log: cận trên 1.6% > baseline 0.0% + 1%), R3 (braking slice: cận trên 7.3% > baseline 0.0% + 5%), R4 (scenario grid: 16.1% > baseline 0.0% + 2%), và R6 (reactive world: 21.2% > baseline 5.1% + 2%). Hai nguyên nhân gốc rễ bao gồm: hiện tượng trôi dạt phân phối (covariate shift) khi tự lái vòng kín khiến sai số tích luỹ đưa xe vào các trạng thái lạ, và sự thiếu hụt các kịch bản tạt đầu cự ly hẹp trong log tự nhiên. Giải pháp sửa đổi kết hợp bổ sung 20,000 kịch bản cắt đầu giả lập (197,170 cửa sổ) và 2 vòng lặp thuật toán DAgger (mở rộng tập huấn luyện lên 895,211 cửa sổ), ghi nhận chính các trạng thái chệch hướng do mô hình tự đi vào và gán nhãn chuẩn từ chuyên gia IDM. Sau khi sửa, PercMLP+ giảm tỷ lệ va chạm trên lưới cắt đầu từ 16.1% xuống 0.5%, va chạm vòng kín trên log giảm còn 0.3% (cận trên 0.6% <= 1.0%), vượt qua toàn bộ 7/7 quy tắc và được cấp quyền RELEASE. Tuy nhiên, PercMLP+ hoàn toàn KHÔNG tốt hơn IDM (IDM giữ 0.0% va chạm trên lưới, trong khi PercMLP+ vẫn đâm 0.5% ở seed xấu nhất; trên tốc độ 20 m/s chưa từng thấy, PercMLP+ vẫn đâm 0.4% .. 4.4% trong khi IDM là 0.0%). PercMLP+ chưa nên được đưa ngay vào sản xuất thương mại; tôi đề xuất cần bổ sung tầng bảo vệ an toàn cứng (Safety Shielding / AEB fallback) và kiểm thử trên sa hình thực tế trước khi phát hành.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Mặc dù đường ống kiểm định đã giải quyết nhiều bài toán cốt lõi, kết luận phát hành trong bài lab này vẫn bị giới hạn bởi ba yếu tố then chốt:
1. Giả định động học một chiều (1D Longitudinal Dynamics): Bài lab giả định xe chỉ di chuyển trên một trục thẳng, hoàn toàn bỏ qua động lực học ngang (lateral steering), góc đánh lái, hiện tượng trượt bánh (tire slip) và biến đổi độ bám đường (ma sát mặt đường khô, ướt, băng tuyết). Trong thực tế, va chạm giao thông thường gắn liền với thao tác đánh lái tránh chướng ngại vật hoặc trượt rê ngang xe.
2. Mô hình cảm biến và độ trễ lý tưởng hóa: Hệ thống mô phỏng chỉ đưa vào độ trễ cố định (0.6s) và nhiễu Gaussian đơn giản, trong khi hệ thống Perception ngoài đời thực (Camera, LiDAR, Radar) thường xuyên gặp hiện tượng góc khuất (occlusion), mục tiêu ảo (ghost targets), mất dấu tracking khi thời tiết xấu và độ trễ tính toán dao động thất thường (jitter).
3. Thiếu tính tương tác đa tác nhân phức tạp (Multi-agent Game Theory): Môi trường thử nghiệm chỉ gói gọn trong 3 xe thẳng hàng với các tham số IDM cơ bản, không phản ánh được tính phức tạp của giao thông đô thị nơi có sự xuất hiện của xe máy, người đi bộ, xe buýt, các nút giao cắt đồng mức và sự tương tác tâm lý khó lường giữa các tài xế.

Để thu hẹp khoảng cách tới triển khai thực tế, bước đi tiếp theo là chuyển đổi mô hình sang các nền tảng mô phỏng 3D đa vật lý (như CARLA, SUMO), tiến hành kiểm thử Hardware-in-the-Loop (HIL) trên sa hình đóng (Proving Ground), và xây dựng kiến trúc điều khiển hai tầng (Dual-layer Control) kết hợp mạng nơ-ron học máy với thuật toán kiểm soát an toàn tất định (Control Barrier Functions hoặc Safety Fallback bằng IDM/AEB).


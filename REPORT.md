# REPORT — Lab Day 12

| | |
|---|---|
| Họ và tên | Phạm Văn Kiên |
| MSSV (đúng bằng `--student-id` đã chạy) | 2A202602590 |
| GitHub | [pvksssss](https://github.com/pvksssss) |
| Repository Public | [K4-Track4-DAY12-PhamVanKien-2A202602590](https://github.com/pvksssss/K4-Track4-DAY12-PhamVanKien-2A202602590) |
| Seed chia dữ liệu / seed mô hình | 7509 / 9112, 9113, 9114, 9115, 9116 |
| Số seed huấn luyện | 5 |

Lệnh tạo bằng chứng: `python run_all.py --student-id 2A202602590` (không dùng `--quick`). Môi trường: Python 3.11.9, NumPy 2.4.6, pandas 3.0.6, matplotlib 3.11.2; giới hạn BLAS/OMP ở 1 luồng. Mã chỉ thay thân 10 hàm TODO; giữ nguyên bộ kiểm thử, mô phỏng và ngưỡng cổng.

## Kết quả cuối

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---:|---:|---|
| constant-velocity | 1,2524 | 20,2567 | BLOCK |
| IDM | 1,5048 | 0,1387 | baseline |
| EgoMLP | 0,7572–0,7623 | 12,1054–25,4249 | BLOCK |
| PercMLP | 0,6174–0,6278 | 0,8325–3,2258 | BLOCK |
| PercMLP+ | 0,7957–0,8492 | 0,3122–0,3815 | RELEASE (7/7) |

Các mô hình học nêu khoảng nhỏ nhất–lớn nhất qua 5 seed. Nguồn: [openloop.json](evidence/openloop.json), [closedloop.json](evidence/closedloop.json), [gate.json](evidence/gate.json); L2 từng seed được tính lại trực tiếp từ các model đã lưu trong [supplemental_metrics.json](evidence/supplemental_metrics.json). RELEASE là kết luận của cổng bài lab, không phải phê duyệt triển khai xe thật.

## Câu 1 · Dữ liệu

Theo [data_audit.json](evidence/data_audit.json), dữ liệu có 992.679 hàng và 9.993 quỹ đạo; loại 383 bản trùng còn 9.610 quỹ đạo.
Tương quan giữa `Acc_LV` và đạo hàm tốc độ xe dẫn chỉ 0,0454, trong khi cột tương ứng của xe theo sau đạt 0,9291: không thể tin trực tiếp nhãn gia tốc xe dẫn để tìm ca phanh.
Thay thế bằng `a_lead`: làm mượt `Speed_LV` bằng trung bình trượt 5 mẫu rồi lấy đạo hàm theo thời gian, với bước 0,1 s; cách này giảm việc nhận nhầm nhiễu tốc độ thành phanh.
Loại trùng trước khi chia giúp tránh cùng một diễn biến xuất hiện ở cả train và test dù ID khác nhau.
Chia theo toàn bộ quỹ đạo với seed 7509, giữ lại 2.883 quỹ đạo test; chia ngẫu nhiên từng frame sẽ làm các cửa sổ gần như giống nhau lọt sang hai tập và thổi phồng khả năng tổng quát hóa.

## Câu 2 · Phát lại

[replay.json](evidence/replay.json) ghi nhận chạy cùng mã hai lần lệch 0/600 đoạn; refactor float32 khác giá trị chính xác ở 600/600 đoạn nhưng 0 đoạn vượt `atol=rtol=0,001`, sai khác lớn nhất khoảng 0,00012 m/s².
Đổi thứ tự phép tính và độ chính xác số thực làm thay đổi sai số làm tròn; dùng tiêu chuẩn `|new−golden| ≤ atol + rtol×|golden|` phù hợp hơn đòi hỏi bằng nhau tuyệt đối đối với lệnh gia tốc của bài lab.
AEB làm lệch 6/600 đoạn. Đã mở đoạn 140: các frame 34–40 đổi lệnh, chi tiết được tính từ log và lưu trong [replay_aeb_detail.json](evidence/replay_aeb_detail.json).
Ví dụ frame 34 có gap 4,4608 m, tốc độ tiến gần 3,007 m/s, TTC 1,4835 s < 1,5 s, nên lệnh đổi từ −5,3258 sang −6 m/s²; nếu IDM đã phanh mạnh hơn −6 thì AEB không đổi lệnh dù TTC thấp.
Phát nhanh nhất có 0 frame trễ/rớt; chế độ theo dấu thời gian có 581 frame trễ và 570 frame rớt trên 59.524 frame, làm 376/600 đoạn thay đổi.
Đây là chi phí xử lý và lịch đến thông điệp được mô phỏng, cho thấy lỗi deadline bị che khi bỏ ràng buộc thời gian; không phải phép đo độ trễ phần cứng thật.

## Câu 3 · Kịch bản

Theo [scenarios.json](evidence/scenarios.json), lưới gồm 60 khoảng cách × 45 tốc độ tiến gần = 2.700 ô; tốc độ cruise 12 m/s, gap 2,7601–59,3123 m, tốc độ tiến gần 0–12 m/s.
Biên suy từ quãng đường mất trong thời gian trễ và khi phanh: `gap < dv×t_react + dv²/(2×9)` với `dv>0`; đây là vùng không bộ điều khiển nào tránh được dưới giả định xe dẫn giữ tốc độ.
Vùng bất khả thi tăng từ 2,2222% lên 7,4444% khi độ trễ tăng từ 0 lên 0,6 s; vùng đâm của IDM tăng tương ứng 5,2222 điểm phần trăm, trong khi diện tích tránh được nhưng vẫn đâm của IDM bằng 0 ở cả hai độ trễ.
FullBrake khớp biên với 0 ô lệch ngoài vùng cách biên dưới 5% ở cả hai lần kiểm chứng; vì vậy không quy tất cả va chạm cho lỗi điều khiển.
Khai thác được 614 ca phanh, chiếm 6,3892% quỹ đạo; mức phanh người lái ở phân vị cực trị được dùng trong bài là 3,6248 m/s², thấp hơn giới hạn cơ cấu 9 m/s².
Có 8,5556% lưới đòi phanh mạnh hơn mức đó. Các ca log trong hình [scenarios_idm.png](evidence/scenarios_idm.png) chưa phủ đều vùng gap nhỏ và tốc độ tiến gần lớn, nên bắt chước log có thể bỏ sót các trạng thái khó của xe cắt vào.

## Câu 4 · Vòng hở và vòng kín

Bảng dưới dùng L2 trung bình và tỷ lệ va chạm trung bình qua seed để xếp hạng đúng như `closedloop.json/ranking`; cổng phát hành dùng seed tệ nhất, còn bảng kết quả cuối nêu cả dải seed.

| Bộ điều khiển | L2 trung bình (m) | Hạng vòng hở | Va chạm trung bình (%) | Hạng vòng kín |
|---|---:|---:|---:|---:|
| constant-velocity | 1,2524 | 3 | 20,2567 | 4 |
| IDM | 1,5048 | 4 | 0,1387 | 1 |
| EgoMLP | 0,7589 | 2 | 19,6531 | 3 |
| PercMLP | 0,6222 | 1 | 1,3736 | 2 |

Ba cặp đảo thứ tự là IDM/constant-velocity, IDM/EgoMLP và IDM/PercMLP, theo [closedloop.json](evidence/closedloop.json).
IDM không tối ưu bắt chước đúng quỹ đạo người lái: nó ưu tiên khoảng cách và phản ứng theo tốc độ tương đối, nên sai số dự đoán cao hơn vẫn có thể đi cùng khả năng tránh va chạm tốt hơn.
EgoMLP chỉ thấy lịch sử tốc độ bản thân, thiếu gap và tốc độ xe dẫn; trên log nó có thể ngoại suy quán tính khá tốt nhưng không biết lúc nào cần phanh vì xe phía trước.
Ở vòng kín, sai số lệnh thay đổi trạng thái đầu vào tiếp theo và tích lũy; phân bố trạng thái khác với các cửa sổ người lái trong train, nên L2 thấp không bảo đảm tự lái tốt.
[openloop.json](evidence/openloop.json) còn cho thấy lát cắt sắp phanh chỉ chiếm 4,4004% cửa sổ: L2 PercMLP tăng từ 0,6222 lên 0,8684 m và EgoMLP từ 0,7589 lên 1,0720 m trên lát cắt, điều bị điểm trung bình che bớt.

## Câu 5 · Một lần chạy chưa đủ

Theo `closedloop.json/logs/EgoMLP/per_seed`, cùng 2.883 quỹ đạo test, EgoMLP dao động từ 12,1054% đến 25,4249%, chênh 13,3195 điểm phần trăm.
Seed 9115 tốt nhất có 349 va chạm, Wilson 95% [10,9649%; 13,3469%]; seed 9113 tệ nhất có 733 va chạm, khoảng [23,8688%; 27,0464%].
Hai khoảng không chồng nhau; dao động do huấn luyện lớn hơn nhiều mức bất định lấy mẫu thể hiện trong từng khoảng.
Wilson mô tả bất định tỷ lệ dưới giả định lấy mẫu Bernoulli cho một mô hình cố định, không thay thế dải qua seed và cũng không chứng minh an toàn ở phân bố khác.
Chỉ chọn seed 9115 có thể báo 12,1054%, thấp hơn constant-velocity 20,2567%; chọn seed 9113 lại có thể báo 25,4249%, cao hơn baseline đó và đảo kết luận so sánh.
Vì vậy báo đủ 5 seed, khoảng tin cậy, trung bình và trường hợp tệ nhất; R5 của cổng giới hạn dải ở 1 điểm phần trăm, mà EgoMLP không đạt.

## Câu 6 · Tác nhân phản ứng

[agents.json](evidence/agents.json) chọn `T=1,4 s` vì khoảng cách Wasserstein-1 giữa phân bố time headway mô phỏng và người lái nhỏ nhất, 0,3458 so với 0,9025 của xe không phản ứng; trung vị người lái là 1,8589 s.
Xe sau dùng khoảng cách ban đầu 18,8 m và độ trễ 0,6 s. Bảng dưới là phần trăm toàn lưới xảy ra ít nhất một va chạm, lấy kết quả seed tệ nhất theo thế giới phản ứng như mã đánh giá; chỉ nêu 4 bộ điều khiển ban đầu để phân tích tác nhân.

| Bộ điều khiển | Xe sau phát lại, trễ ego 0 s (%) | Xe sau phản ứng, trễ ego 0 s (%) | Xe sau phát lại, trễ ego 0,6 s (%) | Xe sau phản ứng, trễ ego 0,6 s (%) |
|---|---:|---:|---:|---:|
| constant-velocity | 77,6296 | 77,6296 | 77,6296 | 77,6296 |
| IDM | 75,2593 | 2,2222 | 75,7407 | 7,4444 |
| EgoMLP | 77,2222 | 77,2222 | 77,3704 | 77,3704 |
| PercMLP | 80,4074 | 17,2593 | 77,2222 | 24,5185 |

Ở trễ ego 0,6 s, IDM bị đâm phía sau trên 68,2963% lưới nếu xe sau cứ chạy theo log, nhưng bằng 0% nếu xe sau phản ứng; đó là hậu quả thế giới phát lại không biết ego đã phanh.
Theo tổng va chạm, IDM nhìn chỉ nhỉnh hơn constant-velocity một chút trong thế giới phát lại (75,7407% so với 77,6296%), nhưng khác biệt rất lớn khi xe sau phản ứng (7,4444% so với 77,6296%).
PercMLP thậm chí trông kém constant-velocity tại trễ 0 s khi xe sau phát lại, rồi tốt hơn khi xe sau phản ứng; vì vậy không thể đánh giá chính sách phanh mà bỏ tương tác với xe phía sau.
Hiệu chỉnh time headway giúp tác nhân hợp lý hơn theo một phân bố đã đo; nó không chứng minh mô hình IDM bao quát hết cách người lái thật phản ứng.

## Câu 7 · Cổng phát hành và sửa

[gate.json](evidence/gate.json) chặn constant-velocity bởi R2, R3, R4, R6; chặn EgoMLP và PercMLP bởi R2, R3, R4, R5, R6. Tất cả ứng viên đạt R1 và R7: replay tái lập được, có 614 sự kiện và 2.700 ô lưới.
Ở PercMLP, R4/R6 chỉ ra khoảng trống bao phủ xe cắt vào; R2/R3 phản ánh sai số điều khiển tích lũy và lỗi trên ca phanh, còn R5 cho thấy độ nhạy seed. Các nguyên nhân này phù hợp với bằng chứng, nhưng bài chưa có ablation tách riêng tác dụng từng cách sửa.
Thêm 20.000 kịch bản cắt vào sinh 197.370 cửa sổ gắn nhãn IDM để phủ trạng thái ngoài log; 2 vòng DAgger/seed cho mô hình tự lái trên tập train rồi học dự báo IDM 3 s ở chính trạng thái nó đi vào, giúp phục hồi khi lệch khỏi trạng thái người lái.

| Quy tắc | PercMLP trước sửa | PercMLP+ sau sửa | Ngưỡng giữ nguyên |
|---|---:|---:|---|
| R2: cận trên Wilson 95% trên log (%) | 3,9356 | 0,6820 | ≤ 0,1387 + 1 điểm % |
| R3: cận trên Wilson 95% trên lát cắt phanh (%) | 22,4688 | 4,4697 | ≤ 1,5544 + 5 điểm % |
| R4: diện tích tránh được nhưng vẫn đâm, trễ 0,6 s (%) | 17,0741 | 1,3704 | ≤ 0 + 2 điểm % |
| R5: dải tỷ lệ va chạm qua seed (điểm %) | 2,3933 | 0,0694 | ≤ 1 điểm % |
| R6: tổng va chạm với xe sau phản ứng (%) | 24,5185 | 8,8148 | ≤ 7,4444 + 2 điểm % |

R2–R6 đều chuyển từ trượt sang qua; PercMLP+ đạt 7/7 quy tắc, `fix.json` và `gate.json` cùng kết luận RELEASE. Va chạm trên log giảm từ 0,8325–3,2258% xuống 0,3122–0,3815% qua seed.
L2 trung bình lại tăng từ 0,6222 lên 0,8216 m, cho thấy bắt chước quỹ đạo người lái tốt hơn không đồng nghĩa kiểm soát an toàn hơn; mục tiêu học sau sửa có thêm hành vi của chuyên gia IDM.
PercMLP+ học từ IDM và không được chứng minh tốt hơn thầy: IDM chỉ có 4/2.883 va chạm, còn PercMLP+ có 9–11; diện tích đâm có thể tránh ở lưới chính là 0% so với tối đa 1,3704%. Cổng chỉ kiểm tra không-kém-hơn trong biên cho phép.
Tại cruise 20 m/s chưa thấy trong dữ liệu cắt vào huấn luyện (tối đa 16 m/s), [fix.json](evidence/fix.json) ghi PercMLP+ vẫn đâm ở 0,2963–2,2222% diện tích tránh được, IDM là 0%; vì vậy chỉ chấp nhận qua cổng lab, chưa thay IDM để phát hành thực tế, và cần mở rộng kiểm thử ngoài phân bố cùng thử nghiệm kiểm soát.

## Câu 8 · Bài lab này không chứng minh được gì

Thứ nhất, mô phỏng chỉ có chuyển động dọc với bước 0,1 s và xe dài cố định 4,5 m: thiếu đổi làn, giao lộ, đường cong và người đi bộ, nên kết luận không áp dụng cho toàn bộ chức năng tự lái.
Thứ hai, đầu vào lấy trực tiếp từ log; phép thử nhiễu chỉ thêm sai số gap 1 m và tốc độ xe dẫn 0,5 m/s theo [closedloop.json](evidence/closedloop.json), chưa kiểm tra che khuất, mất phát hiện hay sai đồng bộ cảm biến ngoài đời. Cần kiểm thử chuỗi cảm biến đến điều khiển và lỗi có tương quan theo thời gian.
Thứ ba, xe dẫn trên log không phản ứng với ego, xe sau chỉ là IDM hiệu chỉnh theo headway. Hai thế giới trong [agents.json](evidence/agents.json) cho kết quả rất khác, nên cần nhiều họ tác nhân, hiệu chỉnh bằng tương tác thật và kiểm tra độ nhạy hành vi.
Thứ tư, test chỉ gồm 2.883 quỹ đạo ngắn và lưới 2.700 ô với một mức cruise chính 12 m/s; không đại diện cho tần suất tai nạn thực tế hay các tình huống hiếm. Cần tập test độc lập rộng hơn về tốc độ, thời tiết, độ bám và kịch bản, đồng thời tránh dùng lặp lại một tập để tối ưu.
Thứ năm, Wilson 95% và 5 seed chỉ mô tả một phần bất định trong phạm vi đã chạy; cổng cho phép các biên không-kém-hơn, không yêu cầu rủi ro bằng 0 và không đo đầy đủ độ êm, tiến độ hay tính hợp lệ giao thông.
Vì vậy cần thêm kiểm thử phần cứng trong vòng lặp, độ trễ thực, giới hạn cơ cấu và đường thử kiểm soát trước khi cân nhắc triển khai; vượt cổng bài lab chỉ là một mốc bằng chứng cho phạm vi mô phỏng.

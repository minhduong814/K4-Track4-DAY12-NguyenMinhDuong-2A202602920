# REPORT — Lab Day 12

> Kết quả chạy đầy đủ với MSSV `2A202602920`, không dùng `--quick`. Số liệu lấy từ các file tự sinh trong `evidence/`; phần mô phỏng 3D chỉ dùng để quan sát.

| | |
|---|---|
| Họ và tên | `Nguyễn Minh Dương` |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `2A202602920` |
| GitHub | [minhduong814](https://github.com/minhduong814/K4-Track4-DAY12-NguyenMinhDuong-2A202602920) |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | `939` / `9541, 9542, 9543, 9544, 9545` |
| Số seed huấn luyện bạn dùng (`--seeds`) | `5` |

## Kết quả cuối

L2 vòng hở và va chạm trên log của mô hình học báo **min–max qua các seed**; bộ xác định chạy một lần. Quyết định cổng xét seed xấu nhất theo từng tiêu chí, không chọn seed thuận lợi.

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1,24993 | 19,94450 | BLOCK — R2, R3, R4, R6 |
| IDM | 1,53943 | 0,06937 | baseline |
| EgoMLP | 0,75491–0,76424 | 14,60284–33,81894 | BLOCK — R2, R3, R4, R5, R6 |
| PercMLP | 0,62473–0,63170 | 0,45092–2,35865 | BLOCK — R2, R3, R4, R5, R6 |
| PercMLP+ (sau khi sửa) | 0,80822–0,84767 | 0,06937–0,24280 | RELEASE trong phạm vi cổng lab — qua 7/7 quy tắc |

Nguồn bảng: [openloop.json](evidence/openloop.json), [closedloop.json](evidence/closedloop.json), [gate.json](evidence/gate.json); L2 từng seed của bản sửa được tính lại từ model đã lưu trong [fix_openloop_per_seed.json](evidence/fix_openloop_per_seed.json), khớp trung bình của script gốc.

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Dữ liệu có 992.679 hàng, 9.993 quỹ đạo gốc; loại 383 bản trùng còn 9.610 quỹ đạo, dài 70–109 bước ở chu kỳ 0,1 giây.\
Tương quan `Acc_LV` với đạo hàm `Speed_LV` chỉ 0,04535, trong khi cột tương ứng của xe theo sau đạt 0,92909; vì vậy không tin `Acc_LV` để khai thác ca phanh.\
Thay bằng tốc độ xe dẫn được làm mượt trung bình trượt 5 mẫu, rồi lấy gradient theo thời gian; yêu cầu gia tốc dưới −2 m/s² ít nhất 5 bước liên tiếp để loại nhiễu đơn lẻ.\
Nếu giữ quỹ đạo trùng ở hai tập, mô hình có thể gặp lại chính tình huống đã học khi kiểm tra; ngay cả các khung lân cận trong cùng quỹ đạo cũng chia sẻ lịch sử và tương lai.\
Vì vậy loại trùng trước, rồi chia theo toàn bộ quỹ đạo bằng seed 939 để train/test độc lập theo định danh quỹ đạo. Nguồn: [data_audit.json](evidence/data_audit.json), `lab12/data.py`.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Chạy lại cùng mã: 0/600 đoạn khác; refactor float32: cả 600 đoạn không giống từng bit nhưng 0 đoạn vượt dung sai, sai khác tuyệt đối lớn nhất khoảng 9,7 × 10⁻⁵ m/s².\
Đổi thứ tự phép tính và độ chính xác làm thay đổi làm tròn; dùng tiêu chuẩn `|new − golden| ≤ 0,001 + 0,001 × |golden|` m/s², coi hai NaN đệm cùng vị trí là bằng nhau. Dung sai này phù hợp phép kiểm tra hồi quy lệnh trong lab, không phải chứng nhận an toàn.\
AEB làm lệch 8/600 đoạn. Mở đoạn 184 thấy khung 4 (0,4 s): gap 4,3682 m, tốc độ tiến gần 2,9339 m/s, TTC 1,48887 s < ngưỡng 1,5 s; lệnh đổi từ −5,61843 sang −6 m/s².\
Chỉ khung đủ điều kiện TTC thấp và lệnh IDM chưa phanh tới −6 mới thực sự đổi đầu ra; các khung khác không nhất thiết lệch dù AEB được bật.\
Trong 59.606 khung, phát nhanh nhất có 0 trễ/0 rớt, còn chế độ thời gian thật mô phỏng chi phí xử lý có 582 trễ và 575 rớt, làm 375 đoạn vượt dung sai: khi quá hạn, hệ thống giữ lệnh cũ. Đây là mô phỏng deadline, không phải đo thời gian xử lý thực trên phần cứng.\
Nguồn: [replay.json](evidence/replay.json), [replay_aeb_detail.json](evidence/replay_aeb_detail.json).

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới cruise 10 m/s gồm 60 khoảng cách từ 2,76005 đến 59,31233 m và 45 tốc độ tiến gần từ 0 đến 10 m/s, tổng 2.700 ô.\
Trong thời gian trễ xe mất `dv × t_react` khoảng cách; phanh tối đa 9 m/s² cần thêm `dv²/(2 × 9)`. Với `dv > 0`, gap nhỏ hơn tổng này là bất khả thi dù điều khiển hoàn hảo.\
Độ trễ 0 → 0,6 s làm phần bất khả thi tăng 0,96296% → 5,14815%, tức 4,18519 điểm phần trăm; vùng va chạm của IDM tăng đúng như vậy, còn vùng tránh được nhưng IDM vẫn đâm là 0% ở cả hai độ trễ.\
FullBrake khớp biên vật lý với 0 ô sai ở xa biên cho cả hai độ trễ; điều này kiểm chứng tính nhất quán của mô phỏng với giả định vật lý đang dùng.\
Khai thác được 614 ca phanh (6,38918% quỹ đạo); mức phanh người lái theo phân vị 99,9 là 3,62484 m/s², thấp hơn giới hạn cơ cấu chấp hành 9 m/s²; 5,22222% lưới đòi phanh mạnh hơn mức phân vị này ngay cả khi chưa thêm trễ. Đây là mốc phân vị, không phải cực đại tuyệt đối của log.\
Các chấm trắng tập trung ở tốc độ tiến gần thấp và khoảng cách vừa phải; vùng gap nhỏ, tiến gần nhanh sát biên nguy hiểm có ít ca log. Nguồn: [scenarios.json](evidence/scenarios.json), [scenarios_idm.png](evidence/scenarios_idm.png).

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Bảng xếp hạng dùng trung bình qua 5 seed cho mô hình học, đúng cách tính `closedloop.json["ranking"]`; L2 là trung bình lỗi vị trí tại 1, 2 và 3 giây, đơn vị m.

| Bộ điều khiển | L2 trung bình (m) | Hạng L2 | Va chạm trung bình (%) | Hạng vòng kín |
|---|---:|---:|---:|---:|
| constant-velocity | 1,24993 | 3 | 19,94450 | 3 |
| IDM | 1,53943 | 4 | 0,06937 | 1 |
| EgoMLP | 0,75939 | 2 | 21,22789 | 4 |
| PercMLP | 0,62757 | 1 | 1,16545 | 2 |

Có 4 cặp đảo thứ tự: constant-velocity/IDM, constant-velocity/EgoMLP, IDM/EgoMLP và IDM/PercMLP; vì vậy L2 tốt hơn không đảm bảo tỷ lệ va chạm thấp hơn.\
IDM chủ động giữ khoảng cách, phanh theo gap và tốc độ tương đối nên không nhất thiết bắt chước đúng quỹ đạo người lái; dự báo vòng hở còn giả định xe dẫn giữ tốc độ. Lệch nhãn người lái có thể tăng L2 dù hành vi vòng kín vẫn an toàn hơn trong các bài thử này.\
EgoMLP chỉ thấy lịch sử tốc độ bản thân, không thấy gap hay xe dẫn đang chậm lại; cùng lịch sử của nó có thể tương ứng những nguy cơ phía trước rất khác nhau, và lỗi nhỏ tiếp tục tích lũy khi tự điều khiển.\
Lát cắt xe dẫn sắp phanh chỉ chiếm 4,88896% trong 71.549 cửa sổ test; L2 EgoMLP/PercMLP tăng từ 0,75939/0,62757 lên 1,07300/0,86582 m trên lát cắt, cho thấy trung bình che mất tình huống khó. Nguồn: [openloop.json](evidence/openloop.json), [closedloop.json](evidence/closedloop.json), [hình so sánh](evidence/openloop_vs_closedloop.png).

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Trên cùng 2.883 quỹ đạo test, EgoMLP dao động 14,60284–33,81894% va chạm, chênh 19,21609 điểm phần trăm, dù L2 qua seed chỉ nằm trong 0,75491–0,76424 m.\
Seed tốt nhất 9543 có 421/2.883 ca đâm, Wilson 95% [13,36088%; 15,93901%]; seed xấu nhất 9541 có 975/2.883 ca đâm, khoảng [32,11453%; 35,56641%]. Hai khoảng không chồng nhau.\
Wilson biểu diễn bất định của tỷ lệ trên tập tình huống hữu hạn cho từng mô hình; dải giữa seed biểu diễn thêm sự nhạy cảm với quá trình học. Hai nguồn biến thiên này không thay thế nhau và cũng không phải bảo đảm cho mọi tình huống ngoài mẫu.\
Nếu chỉ chọn seed 9543, có thể báo 14,60% và kết luận tốt hơn constant-velocity (19,94%); chọn seed 9541 lại báo 33,82% và kết luận ngược lại. Trung bình đủ 5 seed là 21,22789%, xếp dưới constant-velocity.\
Vì vậy phải báo cả dải, khoảng tin cậy từng seed và quy tắc lấy seed xấu nhất ở cổng; không chọn seed có lợi để biện minh phát hành. Các seed dùng chung tập test nên không cộng số mẫu thành 5 × 2.883 quan sát độc lập. Nguồn: [closedloop.json](evidence/closedloop.json), [openloop.json](evidence/openloop.json).

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Chọn `T = 1,2 s` vì khoảng cách Wasserstein giữa phân bố headway mô phỏng và người lái nhỏ nhất: 0,35251 so với 0,87980 của xe không phản ứng; headway trung vị người lái là 1,82949 s. Đây là khớp một thống kê hành vi, chưa chứng minh xe sau giống người thật trong mọi cảnh.\
Bảng dưới là tỷ lệ % trên lưới khi ego trễ 0,6 s; xe sau cách 14 m và phản ứng trễ 0,6 s. Với mô hình học, mỗi hàng lấy seed có tỷ lệ **any trong thế giới phản ứng** cao nhất, đúng cách tổng hợp `agents.json`.

| Bộ điều khiển | Phát lại: trước (%) | Phát lại: sau (%) | Phát lại: bất kỳ (%) | Phản ứng: trước (%) | Phản ứng: sau (%) | Phản ứng: bất kỳ (%) |
|---|---:|---:|---:|---:|---:|---:|
| constant-velocity | 73,37 | 0,00 | 73,37 | 73,37 | 0,00 | 73,37 |
| IDM | 5,15 | 66,63 | 71,78 | 5,15 | 0,00 | 5,15 |
| EgoMLP | 77,00 | 0,00 | 77,00 | 77,00 | 0,00 | 77,00 |
| PercMLP | 23,74 | 52,22 | 75,63 | 25,44 | 0,00 | 25,44 |

IDM phanh để tránh xe cắt vào nhưng xe sau phát lại giữ nguyên tốc độ nên đâm vào ego ở 66,62963% ô; tác nhân phản ứng loại toàn bộ va chạm phía sau này. Khi ego không trễ, con số tương ứng còn là 70,40741% so với 0%.\
Trong thế giới phát lại, IDM trông chỉ nhỉnh hơn constant-velocity (71,78% so với 73,37%); khi xe sau phản ứng, lợi thế rõ hơn nhiều (5,15% so với 73,37%). Cặp này không đổi hạng trong số liệu của tôi; cặp PercMLP/constant-velocity mới thực sự đổi hạng giữa hai thế giới.\
Không thể quy hết các cú đâm sau do giả định “không phản ứng” cho lỗi bộ điều khiển ego. Mô phỏng dừng mỗi ô khi va chạm nên va chạm sau còn có thể che một va chạm trước xảy ra muộn hơn, giải thích tỷ lệ trước của PercMLP khác nhau giữa hai thế giới. Nguồn: [agents.json](evidence/agents.json).

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

Trước sửa, constant-velocity rớt R2/R3/R4/R6; EgoMLP và PercMLP rớt R2/R3/R4/R5/R6. Tất cả qua R1 (phát lại tái lập) và R7 (614 ca phanh, 2.700 ô); không thay bất kỳ ngưỡng cổng nào.\
PercMLP rớt R2 vì cận trên Wilson seed xấu nhất 2,97927% > 0,06937% + 1 điểm; R3 vì cận trên ở 195 quỹ đạo có phanh là 13,51686% > 0,51282% + 5 điểm. Sai số bắt chước tích lũy làm mô hình rời những trạng thái người lái đã đi qua; R5 cũng rớt vì dải va chạm 0,45092–2,35865% rộng 1,90773 điểm, vượt mức 1 điểm.\
Nguyên nhân thiếu độ phủ tình huống thể hiện ở R4: 20,29630% **tổng số ô lưới** thuộc vùng tránh được nhưng vẫn đâm, cao hơn baseline 0% + 2 điểm; R6: thế giới xe sau phản ứng có 25,44444% ô va chạm, vượt baseline 5,14815% + 2 điểm. Kết hợp 5,22222% lưới đòi phanh mạnh hơn mốc người lái trong log cho thấy học từ log chưa đủ cho họ cắt vào khó.\
Sửa bằng 20.000 ca cắt vào mô phỏng tạo 197.180 cửa sổ có nhãn IDM, cùng 2 vòng DAgger cho mỗi seed: để mô hình tự lái trên **tập train**, thu trạng thái nó thực sự gặp rồi gắn nhãn dự báo IDM 3 giây tại chính trạng thái đó. Cách đầu bổ sung họ tình huống còn thiếu; cách sau dạy phục hồi khi mô hình tự lệch khỏi phân bố trạng thái người lái.\

Sau sửa, PercMLP+ qua 7/7 quy tắc; R2–R6 chuyển từ rớt sang qua. Cận trên Wilson trên log còn 0,50037% ≤ 1,06937%; ở lát cắt phanh là 2,84724% ≤ 5,51282%. R4 còn 0,18519% tổng số ô tránh được nhưng vẫn đâm, R5 có dải rộng 0,17343 điểm phần trăm, R6 còn 5,33333% ô va chạm trong thế giới phản ứng.\
Va chạm trên log giảm từ 0,45092–2,35865% xuống 0,06937–0,24280%; nhưng L2 trung bình tăng từ 0,62757 lên 0,83025 m. Điểm bắt chước người lái xấu đi trong khi các phép thử vòng kín tốt lên: mục tiêu L2 không tương đương mục tiêu an toàn.\
Qua cổng chỉ cho thấy đáp ứng tiêu chí không kém hơn trong các bài thử này; PercMLP+ học nhãn từ IDM và chưa có bằng chứng tốt hơn IDM (IDM đâm 0,06937% trên log). Học trò có thể khái quát khác thầy hoặc được cải thiện bằng mục tiêu/dữ liệu khác, nhưng bắt chước không tự bảo đảm vượt thầy. Ở cruise 20 m/s, ngoài dải cắt vào huấn luyện tối đa 16 m/s, bản sửa vẫn có 0,07407–1,85185% ô tránh được nhưng đâm, so với IDM 0,00000–0,00000%; cần kiểm tra rộng hơn trước khi thay IDM, chạy shadow và thử trường có giám sát.\
Nguồn sau sửa: [fix.json](evidence/fix.json), [gate.json](evidence/gate.json), [closedloop.json](evidence/closedloop.json), [openloop.json](evidence/openloop.json), [L2 từng seed của bản sửa](evidence/fix_openloop_per_seed.json).

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Thứ nhất, mô phỏng chỉ xét chuyển động dọc và bước thời gian 0,1 s: không kiểm tra điều khiển ngang khi cắt vào, hình học va chạm thật, độ bám đường hay cơ cấu chấp hành thay đổi. Biên vật lý khớp ở 0 và 0,6 s chỉ xác nhận mô hình tính toán này; cần mô phỏng động lực học đầy đủ và kiểm thử trên xe/thử trường có kiểm soát.\
Thứ hai, không có chuỗi cảm biến thực: gap và vận tốc được cấp trực tiếp; nhiễu đơn giản trong bước vòng kín chưa đại diện cho che khuất, mất mục tiêu, nhận nhầm đối tượng hay độ trễ biến thiên. Cần kiểm thử perception đến điều khiển, dữ liệu cảm biến khó và phần cứng trong vòng lặp, đo deadline thật thay vì chỉ mô phỏng chi phí.\
Thứ ba, xe dẫn phát lại log không phản ứng với xe thử; xe sau IDM dù hiệu chỉnh vẫn là giả định hành vi. Chênh lệch hai thế giới xe sau cho thấy chính giả định tác nhân có thể đổi kết luận; cần nhiều mô hình người lái, tương tác đa tác nhân và đối chiếu phản ứng với dữ liệu thực.\
Thứ tư, 614 ca phanh và 2.700 ô lưới chỉ phủ các họ tình huống đã chọn, không phải phân bố giao thông thực hay bằng chứng cho rủi ro cực hiếm. Các seed cùng dùng một tập kiểm tra nên không được xem là các tập dữ liệu độc lập; cần dữ liệu độc lập theo nguồn/cảnh, thêm nhiều họ tình huống và kiểm tra ngoài miền thiết kế.\
Vì vậy qua 7 quy tắc chỉ là đủ điều kiện tiếp tục đánh giá trong phạm vi lab. Trước quyết định phát hành cần xác định miền vận hành, tiêu chí rủi ro theo mức nghiêm trọng/phơi nhiễm, thử trường an toàn, chạy shadow và giám sát có phương án dự phòng; chưa có bằng chứng để tuyên bố hệ thống an toàn ngoài đường.\
Nguồn phạm vi và độ phủ: [data_audit.json](evidence/data_audit.json), [scenarios.json](evidence/scenarios.json), phần “Giới hạn” của [README.md](README.md).

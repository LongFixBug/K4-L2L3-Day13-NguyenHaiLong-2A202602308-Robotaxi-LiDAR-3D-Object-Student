# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Ca Ngày 13 — Cá nhân (Làm lẻ)
- Thành viên: xem `TEAMMATES.md` (Nguyễn Hải Long / MSSV: 2A202602308, hoàn thành độc lập toàn bộ các lượt).
- Trạng thái: `executed-by-group`
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Nguyễn Hải Long; 2026-10-02 16:49 (UTC+7); Windows 11 x86_64, Docker Desktop Linux container (amd64, 4 CPUs, 4GB RAM).
- Image tag và image ID; phiên bản repo: Image tag: `day13-pointpillars:lc-20261001-amd64`; Image ID: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`; Repo revision: `0831856d921609312d42c7582c366e5a311bb7b1`.
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `demo.pcd` (mẫu KITTI 000008 gồm 17,238 points); SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`.
- Checkpoint: PointPillars KITTI có sẵn trong image (`/opt/PointPillars/pretrained/epoch_160.pth`); SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`.
- Phạm vi: front-window (`--from KITTI`); score threshold: `0.3`.
- Giả định kênh thứ tư/intensity và nguồn z_ground: Kênh thứ 4 sử dụng hằng số adapter theo lớp (constant reflectance placeholder, RGB=0); nguồn `z_ground = 0.075m` ước lượng tự động từ phân bố cao độ mặt đất của PCD nguồn.

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/boxes-demo-delta-0-voxel-0.16.json`, `run-A/side-demo-delta-0-voxel-0.16.png`, `run-A/summary.csv` | Chỉ nhận diện được 1 hộp `vehicles` duy nhất (tọa độ x=13.15, y=-0.45, z=0.33), score rất thấp 0.322. Hầu hết các xe phía trước đều bị bỏ sót do delta=0 khiến đám mây điểm nằm ngoài phân bố sensor height của mô hình huấn luyện. |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `run-B/side-demo-delta-1.73-voxel-0.16.png`, `run-B/summary.csv` | Baseline chuẩn. Nhận diện được 13 hộp (10 vehicles, 1 two-wheels, 2 pedestrian). Các xe ở gần và cự ly trung bình đều có score cao (0.80 - 0.93), phân bố cao độ tâm hộp bám sát mặt đường cục bộ. |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `run-C/side-demo-delta-1.73-voxel-0.32.png`, `run-C/summary.csv` | Khi kích thước pillar XY tăng gấp đôi (từ 0.16m lên 0.32m) mà không train lại model, toàn bộ 10 hộp vehicles và 1 hộp two-wheels biến mất; chỉ phát hiện được 6 hộp `pedestrian`. |

- A/B — chỉ đổi delta: A có 1 hộp; B có 13 hộp. Ảnh Side vùng $x \in [3, 45]$m khác rõ rệt ở chỗ lượt B bao phủ toàn bộ chuỗi xe trên làn đường, còn lượt A chỉ bắt được 1 xe mờ nhạt. Đây là chạy lại model trên input khác: phép dịch $z_{model} = z_{source} - z_{ground} - delta$ đưa dữ liệu điểm vào đúng dải không gian mà mạng PointPillars học được trên KITTI (+1.73m là chiều cao đặt sensor), không phải chỉ đơn thuần cộng trừ tọa độ sau khi model xuất hộp; điều em còn chưa chắc là độ chính xác của các hộp ở cự ly xa $>40$m khi không có ảnh camera đối chiếu.
- B/C — chỉ đổi pillar: B có 13 hộp; C có 6 hộp. Ảnh Side và file JSON cho thấy: toàn bộ 10 xe `vehicles` bị biến mất hoàn toàn, xuất hiện 6 hộp gán nhãn `pedestrian` với kích thước nhỏ. Số lượng và lớp thay đổi mạnh mẽ do việc tăng kích thước cột (pillar) lên gấp đôi làm giảm độ phân giải lưới không gian $x-y$, khiến các đặc trưng hình học footprint của ô tô bị làm thô (gộp điểm vào các cột lớn), phá vỡ pattern nhận dạng của feature extractor. Không có đủ bằng chứng để kết luận C tốt hơn; thực tế C kém hơn nhiều vì gây miss toàn bộ vật thể ô tô.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?: Cấu hình `--from KITTI` giới hạn vùng quan sát cửa sổ phía trước (front ROI, $x > 0$), do đó các đối tượng ở phía sau hoặc ngoài biên không được coi là model bỏ sót (miss). Góc nhìn Side là hình chiếu trực giao $x-z$ (bị nén/chồng lấn trục $y$), nên nhiều xe ở các làn khác nhau có thể bị đè lên nhau trên một mặt phẳng, và hướng xoay (yaw) của xe không thể kiểm tra được ở góc Side mà bắt buộc phải kết hợp góc nhìn Trên (Top view) và ảnh camera.
- JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?: Cả 3 file JSON này đều được chạy trên PCD KITTI demo, tuyệt đối không đủ cơ sở và không được phép import vào các job Robotaxi trên CVAT. Đối với các prediction Robotaxi được nạp từ portal, chúng cũng chỉ là gợi ý ban đầu; cần kiểm tra tiếp cả 5 class của schema (`vehicles`, `two-wheels`, `pedestrian`, `Animal`, `Obstacle`), đối chiếu ảnh camera để xác định đúng đầu xe (yaw), kiểm tra đáy hộp bám mặt đất cục bộ và rà soát hộp thiếu/thừa trước khi nộp.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 m | Không đổi | Kiểm từng hộp | JSON và ảnh `side-correct.png` giữ nguyên prediction gốc từ lượt B; đáy hộp tiếp xúc đúng mặt đất cục bộ. |
| case-batch-z | 13 / 13 | -1.805 m | Không đổi | Dừng batch, kiểm pipeline | Toàn bộ 13/13 hộp trên ảnh `side-batch-z.png` đều bị chìm sâu xuống dưới mặt đất đúng một khoảng bằng $z_{ground} + delta = 0.075 + 1.73 = 1.805$m. Đây là lỗi quên phép biến đổi nghịch đảo khi xuất hộp từ tọa độ model về tọa độ nguồn. Phải dừng sửa tay và yêu cầu fix code transform pipeline. |
| case-one-box-z | 1 / 13 | -1.805 m (chỉ hộp đầu tiên) | Không đổi | Kiểm từng hộp | Chỉ có duy nhất 1 hộp bị chìm xuống dưới mặt đất, 12 hộp còn lại nằm đúng vị trí. Đây là lỗi cục bộ của đối tượng cụ thể (nhiễu điểm hoặc hình học dị biệt), không phải lỗi pipeline chung; do đó cần kiểm tra từng hộp thay vì dừng cả batch. |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

- **Nguyễn Hải Long (MSSV: 2A202602308)**: 
  - **Vai trò đã làm**: Độc lập hoàn thành toàn bộ thí nghiệm (thiết lập môi trường Docker Desktop, chạy runner PointPillars tự động cho cả 3 lượt A/B/C và 3 ca QC, trích xuất dữ liệu JSON/Side/CSV và phân tích kết quả).
  - **Quan sát A/B/C**: Qua đối chiếu `run-A` và `run-B`, nhận thấy việc điều chỉnh `delta=1.73` đưa đám mây điểm về đúng dải cao độ sensor gốc của KITTI, giúp số lượng hộp phát hiện tăng vọt từ 1 lên 13 với confidence score rất cao ($>0.9$ cho các xe ô tô). Ở `run-C`, việc tăng kích cỡ pillar lên $0.32$m làm triệt tiêu khả năng phát hiện ô tô và chỉ ra 6 hộp pedestrian, cho thấy sự nhạy cảm của kiến trúc trích xuất đặc trưng cột đối với độ phân giải voxel.
  - **Diễn giải phép z thuận/ngược**: Trong pipeline KITTI, trước khi đưa điểm vào mô hình cần chuẩn hóa: $z_{model} = z_{source} - z_{ground} - delta$. Khi mô hình xuất hộp dự đoán, bắt buộc phải áp dụng phép biến đổi ngược: $z_{source} = z_{model} + z_{ground} + delta$ để đưa cuboid về đúng hệ quy chiếu của cảm biến nguồn.
  - **Quyết định lỗi batch và hành động**: Nếu phát hiện toàn bộ các hộp trong frame bị dịch chuyển cùng một khoảng chiều cao (như trong `case-batch-z` lệch đồng loạt $-1.805$m), đây là lỗi hệ thống của bước transform, hành động đúng là dừng sửa tay và báo kỹ thuật kiểm tra pipeline. Ngược lại, nếu chỉ có 1 vài hộp cá biệt bị lệch (như `case-one-box-z`), hành động đúng là giữ nguyên pipeline và kiểm tra, tinh chỉnh thủ công từng hộp bằng nhiều góc nhìn.
  - **Điều chưa chắc**: Khi quan sát các vật thể ở rìa ROI hoặc cự ly xa ($>40$m), mật độ điểm phản xạ rất thưa thớt, việc xác định chiều dài thân xe và góc quay yaw nếu chỉ dựa vào LiDAR là rất khó và có độ bất định cao, cần sự hỗ trợ của ảnh camera đồng bộ.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:

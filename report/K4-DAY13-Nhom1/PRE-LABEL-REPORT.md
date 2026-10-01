# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Nhóm 1
- Thành viên: xem `TEAMMATES.md` (Nguyễn Thái Hà - Nhóm trưởng, Nguyễn Hữu Tài, Nguyễn Hoàng Tùng, Vũ Trung Định).
- Trạng thái: `executed-by-group`
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Nguyễn Thái Hà; 2026-10-01 15:05 UTC+7; macOS Apple Silicon (arm64), Docker Desktop Linux VM.
- Image tag và image ID; phiên bản repo: `day13-pointpillars:student`, ID: `sha256:5484cfc43741b14af127616c2e9547a142d2bd8e5163e1a34430562b471bc679`; Git revision: `f5f1de01c98d34240847e3634b4c69fe7376d9db`.
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `data/demo.pcd` (KITTI frame 000008), SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`.
- Checkpoint: PointPillars KITTI epoch 160 (`/opt/PointPillars/pretrained/epoch_160.pth`), SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`.
- Phạm vi: front-window; score threshold: `0.3`
- Giả định kênh thứ tư/intensity và nguồn z_ground: Kênh thứ 4 gán hằng số RGB/placeholder = 0 (do dữ liệu demo lược bỏ reflectance); `z_ground` được ước lượng từ đám mây điểm PCD qua RANSAC/plane fitting.

## Ba lượt inference thật

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/boxes-demo-delta-0-voxel-0.16.json`, `side-demo-delta-0-voxel-0.16.png`, `summary.csv` | Lượt A tạo ra 1 prediction; score của hộp là khoảng 0.322. Số hộp khác lượt B, nhưng không có ground truth để kết luận số đối tượng bị bỏ sót. |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `side-demo-delta-1.73-voxel-0.16.png`, `summary.csv` | Lượt B tạo ra 13 prediction và được dùng làm baseline để so sánh pillar và tạo ca QC; chưa thể xem 13 hộp là số lượng đối tượng đúng. |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `side-demo-delta-1.73-voxel-0.32.png`, `summary.csv` | Tăng pillar từ 0.16m lên 0.32m làm số prediction giảm từ 13 xuống 6. Việc mất đặc trưng hình học là giả thuyết cần kiểm tra thêm bằng các góc nhìn khác. |

- A/B: thay input trước model có khác dịch cùng một hằng số cho output không? Vì sao?
  * **Trả lời:** Việc thay đổi `delta` trước inference làm thay đổi tọa độ điểm đầu vào, quá trình tạo pillar và feature map, nên output có thể thay đổi cả số lượng và vị trí hộp. Ở đây A tạo 1 hộp còn B tạo 13 hộp. Điều này khác với việc lấy output sau inference rồi tịnh tiến cơ học mọi hộp cùng một hằng số 1.73m.
- B/C: thấy gì khi đổi pillar? Có đủ bằng chứng để nói cấu hình nào tốt hơn không?
  * **Trả lời:** Khi tăng pillar từ 0.16m lên 0.32m, số prediction giảm từ 13 xuống 6. Có thể giả thuyết rằng lưới thô hơn làm mất một phần đặc trưng hình học, nhưng chưa đủ bằng chứng để khẳng định B tốt hơn C; cần đối chiếu thêm Top/Side, PCD và camera.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?
  * **Trả lời:** Cấu hình chỉ quét vùng ROI phía trước (front-window), nên vật thể ngoài ROI không được dùng làm bằng chứng miss trong phép so sánh này. Ảnh Side View chiếu x-z và có thể chồng lấp các đối tượng khác y; Side View một mình cũng không đủ để xác minh yaw, nên cần kết hợp Top View và camera.
- JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?
  * **Trả lời:** Cả 3 file JSON đều chưa thể xem là ground truth hoặc import trực tiếp như nhãn cuối. Chúng chỉ là output thực hành; cần đối chiếu từng hộp với PCD và camera đúng frame, rồi kiểm class, tâm, kích thước, yaw và hộp thiếu/thừa trước khi dùng trong workflow gán nhãn.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 m | Không đổi | Chưa kết luận chất lượng | Bản sao không biến đổi của prediction B; không phải ground truth. |
| case-batch-z | 13 / 13 | -1.805 m | Không đổi | **Dừng batch** | Tất cả 13 hộp bị dịch cùng một lượng `delta + z_ground = 1.73 + 0.075 = 1.805 m`; đây là tín hiệu cần kiểm tra phép transform, không sửa tay từng hộp. |
| case-one-box-z | 1 / 13 | -1.805 m | Không đổi | Kiểm từng hộp | Chỉ 1/13 hộp bị dịch; các hộp còn lại không đổi. Đây là ca để kiểm tra cục bộ, chưa tự kết luận nguyên nhân cuối cùng. |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

- **Nguyễn Thái Hà (Nhóm trưởng):**
  * *Vai trò:* Vận hành lệnh Docker lượt A, ghi log/tổng hợp báo cáo lượt B, xem và đối chiếu hình học lượt C.
  * *Quan sát:* Khi chạy lượt A (delta=0), output có 1 prediction trong file `run-A/boxes-demo-delta-0-voxel-0.16.json`, trong khi B có 13 prediction. Kết quả cho thấy output thay đổi khi đổi delta; chưa có ground truth để kết luận số hộp bị bỏ sót.
  * *Hiểu phép đổi z:* Phép đổi z thuận ($z_{model} = z_{source} - z_{ground} - \delta$) chuẩn hóa dữ liệu về hệ trục mà mạng được huấn luyện, sau đó kết quả cần được đổi ngược ($z_{source} = z_{model} + z_{ground} + \delta$) để đưa về hệ tọa độ gốc của xe.
  * *Quyết định:* Trong `case-batch-z`, khi phát hiện 100% hộp bị tụt z, quyết định dừng toàn bộ batch, báo cáo LC để kiểm tra lại pipeline transform, tuyệt đối không chỉnh sửa thủ công từng hộp.
  * *Điều chưa chắc:* Chưa đủ bằng chứng từ output thực hành để xác định các cụm điểm thưa ở vùng biên ROI là đối tượng hay nhiễu; cần kiểm tra đúng PCD và camera nếu có quyền truy cập.

- **Nguyễn Hữu Tài:**
  * *Vai trò:* Kiểm tra cấu hình & JSON lượt A, vận hành lệnh Docker lượt B, ghi log lượt C.
  * *Quan sát:* So sánh `run-B/summary.csv` và `run-C/summary.csv` cho thấy voxel 0.16m tạo 13 prediction, còn 0.32m tạo 6 prediction. Việc lưới thô làm suy giảm độ nhạy là giả thuyết, chưa được xác nhận bằng ground truth.
  * *Quyết định:* Khi gặp lỗi cục bộ như `case-one-box-z`, quyết định giữ nguyên batch và mở viewer 3D để chỉnh sửa riêng hộp bị lỗi.

- **Nguyễn Hoàng Tùng:**
  * *Vai trò:* Xem & đối chiếu hình học lượt A, kiểm tra cấu hình & JSON lượt B, vận hành lệnh Docker lượt C.
  * *Quan sát:* Ảnh chiếu cạnh `side-demo-delta-1.73-voxel-0.16.png` cho thấy các hộp ở vị trí z khác ca batch-z. Đường z=0 trong ảnh chỉ là đường tham chiếu của biểu đồ, nên quan sát này không đủ để khẳng định cấu hình hoặc hình học là chính xác tuyệt đối.
  * *Quyết định:* Nhận thức rõ ràng việc phân biệt lỗi hệ thống (batch error) để tránh lãng phí thời gian gán nhãn thủ công sai lệch.

- **Vũ Trung Định:**
  * *Vai trò:* Ghi log & Báo cáo lượt A, xem & đối chiếu hình học lượt B, kiểm tra cấu hình & JSON lượt C.
  * *Quan sát:* Ảnh `side-batch-z.png` trong `qc-cases` cho thấy toàn bộ 13 hộp bị hạ cùng một lượng xuống dưới đường tham chiếu z=0, minh họa ca lỗi batch-z có kiểm soát.
  * *Điều chưa chắc:* Side View không đủ để xác minh đầy đủ kích thước, yaw hoặc tình trạng che khuất; các thuộc tính đó cần được kiểm tra thêm bằng Top View, PCD và camera khi thực hiện labeling.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:

# Báo cáo bài nộp — Day 23 Sensor Fusion Lab

> Điền file này rồi commit. Cách nộp: [hướng dẫn nộp](../SUBMISSION.md).

## Thông tin học viên

- Họ tên: Nguyễn Trần Nhựt Nam
- MSSV: 2A202602981
- Email: Chưa cung cấp
- Link repo (fork): https://github.com/nhut-nam/K4-L2L3-DAY23-NguyenTranNhutNam-2A202602981-SensorFusion
- Commit hash nộp (`git rev-parse HEAD`): Xem hash của commit CP6 trên `origin/main` khi nộp LMS.

## Tóm tắt kết quả

- `fusion_mode`: `compare`; `frames`: `[0, 198]` (199 frame); `segment`: `training_segment-1005081002024129653_5313_150_5333_150_with_camera_labels.tfrecord`; `seed`: `0`.
- `detection.precision`: `0.9700934579439252`; `detection.recall`: `0.7004048582995951`; `tp/fp/fn`: `519/16/222`.
- `tracking.lidar`: RMSE `0.15032270040636328` m; `matches` `502`; `sum_sq_err` `11.343650957245547` m²; `ghost_track_frames` `0`; `missed_gt_frames` `239`; `mean_confirmed_tracks` `2.522613065326633`.
- `tracking.fused`: RMSE `0.1358667802927652` m; `matches` `502`; `sum_sq_err` `9.26681055753551` m²; `ghost_track_frames` `0`; `missed_gt_frames` `239`; `mean_confirmed_tracks` `2.522613065326633`.
- Fused giảm RMSE khoảng `0.01446` m trên **cùng 502 cặp ghép**. Hai mode đều có `0` ghost và `239` miss, nên kết quả cho thấy camera tinh chỉnh vị trí của các track đã được LiDAR duy trì mà không đổi số track được ghép trong lần chạy này. Coverage theo `matches/det_tp` là `502/519 ≈ 0.967`; precision track theo `matches/(matches+ghosts)` là `1.0`. Camera ở đây lấy tâm hộp ground-truth 2D rồi thêm nhiễu, nên mức cải thiện này không đo chất lượng một camera detector độc lập.

Chạy từ root repo:

```bash
fusion-run-lab --config student/config/paths.yaml --fusion compare --seed 0
```

`rmse = sqrt(sum_sq_err/matches)` trên vị trí 3D của confirmed tracks ghép
một-một với GT xe trong cửa sổ BEV, gate XY **2.0 m**; `null` nếu không có cặp.
Camera dùng tâm hộp 2D ground-truth FRONT có nhiễu seeded, **không** dùng camera
detector. Kết quả này không đo hiệu quả một perception system độc lập với GT.

`grade_run.log` là JSONL, mỗi `(mode,frame)` đúng một record với các trường:
`mode`, `frame`, `det_tp`, `det_fp`, `det_fn`, `valid_gt`, `confirmed`, `matches`,
`sum_sq_err`, `ghosts`, `misses`. Đảm bảo `matches+ghosts==confirmed` và
`matches+misses==valid_gt`; tổng/trung bình record phải khớp `metrics.json`.
File per-mode `metrics_lidar.json`, `metrics_fused.json`, `grade_run_lidar.log`,
`grade_run_fused.log` được giữ để đối chiếu.

## Giải thích ngắn (Parts E–H — tự viết)

1. LiDAR đo tâm hộp 3D bằng `z = [x, y, z]ᵀ` (mét), với `R` 3×3 từ nhiễu theo ba trục. Camera đo tâm hộp ảnh bằng `z = [u, v]ᵀ` (pixel), với `R` 2×2. EKF dùng `h(x)` và Jacobian tương ứng với từng cảm biến.
2. Gating loại cặp track–measurement không hợp lý trước khi gán. Bình phương khoảng cách Mahalanobis `γᵀS⁻¹γ` xét cả độ bất định dự đoán `P` và nhiễu đo `R`; ngưỡng χ² phụ thuộc số chiều đo. Cặp ngoài FOV bị loại trước cả phép chiếu camera.
3. Pipeline là **track-then-fuse**: trong mỗi frame, runtime predict một lần, ghép và update LiDAR, rồi ghép và update camera trên cùng danh sách track. `grade_run.log` có một record mỗi `(mode, frame)` để đối chiếu kết quả LiDAR và fused; nó không chứa từng bước update nội bộ.
4. Calibration camera lệch làm `h(x)` dự đoán sai vị trí pixel, khiến innovation `z − h(x)` có thiên lệch lặp lại theo hướng hoặc vị trí ảnh. Khoảng cách Mahalanobis có thể tăng và bị gate loại; nếu vẫn được nhận, EKF có thể kéo track lệch khỏi vị trí LiDAR.
5. `sensor` tường minh cho biết lượt đang xử lý là LiDAR hay camera ngay cả khi `meas_list` rỗng. Lượt LiDAR rỗng vẫn phải giảm score của track nằm trong FOV và xét xóa. Camera chỉ sửa `x, P` của track ghép được; nó không tạo track, đổi score hoặc xóa track.
6. Track mới có state `initialized`, score `1/window`. LiDAR hit cộng `1/window` (tối đa 1); score **lớn hơn** `confirmed_threshold` thì confirmed. LiDAR miss trong FOV trừ `1/window`, nhưng state confirmed được giữ. Track bị xóa nếu `Pxx` hoặc `Pyy` vượt `max_P`, hoặc confirmed có score dưới `delete_threshold`, hoặc track chưa confirmed có score ≤ 0.

## Bonus (không bắt buộc)

Liệt kê phần bonus đã làm, file bằng chứng trong `student/bonus/` và kết quả chính
(xem [RUBRIC.md](../RUBRIC.md) mục 2). Không làm thì ghi "Không".

- Không.

## Khai báo sử dụng AI (bắt buộc)

Ghi rõ, kể cả khi không dùng ("Không dùng AI"). Xem [RULES.md](../RULES.md) mục 2.

- Công cụ đã dùng (ChatGPT, Copilot, Claude, …): ChatGPT (Codex).
- Dùng cho phần nào (hàm, câu hỏi, debug): Triển khai Part E–H, thiết lập môi trường chạy, giải thích thuật toán và soạn báo cáo này.
- Cách bạn đã kiểm tra lại (pytest, chạy Waymo, đối chiếu công thức): Chạy 128 test trong `student/tests`; chạy Waymo 199 frame ở hai mode với `--seed 0`; đối chiếu metrics với log bằng `check_submission.py`. Người nộp cần đọc và giải thích được mã, công thức và báo cáo trước khi nộp.

## Checklist nộp

- [x] **Part E–H** trong `workspace/` đã implement; `pytest student/tests -q` không còn `failed`/`xfailed` (128 passed; chạy trên Windows với `PYTHONUTF8=1` và `--basetemp .venv/pytest-temp`).
- [x] Part A–D không sửa.
- [x] Lần chạy chấm điểm: `--fusion compare --seed 0`, `frame_start: 0`, `frame_end: 198`.
- [x] Đã commit `student/artifacts/metrics*.json` và `student/artifacts/grade_run*.log` do runner tạo, không sửa tay.
- [x] Đã điền báo cáo kết quả và khai báo AI; địa chỉ email chưa được cung cấp.
- [x] Không commit dữ liệu Waymo, weights, `paths.yaml`, API key.
- [x] `python tools/check_submission.py` báo `KẾT QUẢ: SẴN SÀNG NỘP` sau khi commit CP6.
- [ ] Đã push và nộp link repo + commit hash trên LMS ([hướng dẫn nộp](../SUBMISSION.md)).

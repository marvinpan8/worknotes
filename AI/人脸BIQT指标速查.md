# BIQT 人脸图像质量评估指标速查表

## 一、硬性指标 ⭐

| Column                      | 描述 (Description)     | 建议阈值 / 合格标准  |
| --------------------------- | ---------------------- | -------------------- |
| `quality`                   | 综合质量分数           | **≥ 80**（越高越好） |
| `opencv_face_found`         | 检测到的人脸数量       | **= 1**              |
| `opencv_frontal_face_found` | 检测到的正面人脸数量   | **= 1**              |
| `opencv_profile_face_found` | 检测到的侧面人脸数量   | **= 0**              |
| `opencv_eye_count`          | 检测到的眼睛数量       | **= 2**              |
| `opencv_mouth_count`        | 检测到的嘴巴数量       | **= 1**              |
| `opencv_nose_count`         | 检测到的鼻子数量       | **= 1**              |
| `opencv_landmarks_count`    | 检测到的面部关键点数量 | **≥ 5**              |
| `sap_code`                  | SAP 质量编码           | **≤ 3**（越低越好）  |

## 二、建议指标

| Column          | 描述 (Description) | 建议阈值 / 合格标准      |
| --------------- | ------------------ | ------------------------ |
| `blur`          | 整体图像模糊度     | 越低越好，建议 ≤ 10      |
| `blur_face`     | 人脸区域模糊度     | 越低越好，建议 ≤ 15      |
| `focus`         | 整体图像对焦度     | **≥ 500**（越高越清晰）  |
| `focus_face`    | 人脸区域对焦度     | **≥ 1000**（越高越清晰） |
| `over_exposure` | 整体图像过曝值     | **≤ 1.0**                |
| `over_exposure_face` | 人脸区域过曝值     | **≤ 0.5**  |
| `skin_ratio_face`    | 人脸区域内皮肤占比 | **≥ 0.30** |
| `openbr_confidence`  | OpenBR 检测置信度  | **≥ 0.80** |
| `opencv_face_height` | 人脸高度（像素）   | **≥ 200**  |
| `opencv_face_width`  | 人脸宽度（像素）   | **≥ 200**  |

## 三、辅助指标

| Column         | 描述 (Description)      | 建议阈值 / 合格标准 |
| -------------- | ----------------------- | ------------------- |
| `openbr_IPD`   | OpenBR 瞳孔间距（像素） | ≥ 120               |
| `opencv_IPD`   | OpenCV 瞳孔间距（像素） | ≥ 120               |
| `image_width`  | 图像宽度（像素）        | ≥ 600               |
| `image_height` | 图像高度（像素）        | ≥ 800               |
| `image_area`   | 图像面积（像素²）       | ≥ 480000            |
| `image_channels`               | 图像颜色通道数  | = 3           |
| `image_ratio`                  | 图像宽高比      | 0.7 ~ 1.0     |
| `skin_ratio_full`              | 全图皮肤占比    | ≥ 0.15        |
| `opencv_face_offset_x`         | 人脸水平偏移量  | 绝对值 ≤ 0.15 |
| `opencv_face_offset_y`         | 人脸垂直偏移量  | 绝对值 ≤ 0.15 |
| `opencv_face_center_of_mass_x` | 人脸质心 X 坐标 | 用于计算偏移  |
| `opencv_face_center_of_mass_y` | 人脸质心 Y 坐标 | 用于计算偏移  |
| `opencv_face_x`      | 人脸边界框 X 坐标  | 无直接阈值 |
| `opencv_face_y`      | 人脸边界框 Y 坐标  | 无直接阈值 |
| `openbr_left_eye_x`  | 左眼 X 坐标        | 无直接阈值 |
| `openbr_left_eye_y`  | 左眼 Y 坐标        | 无直接阈值 |
| `openbr_right_eye_x` | 右眼 X 坐标        | 无直接阈值 |
| `openbr_right_eye_y` | 右眼 Y 坐标        | 无直接阈值 |
| `opencv_left_eye_x`  | OpenCV 左眼 X 坐标 | 无直接阈值 |
| `opencv_left_eye_y`    | OpenCV 左眼 Y 坐标 | 无直接阈值 |
| `opencv_right_eye_x`   | OpenCV 右眼 X 坐标 | 无直接阈值 |
| `opencv_right_eye_y`   | OpenCV 右眼 Y 坐标 | 无直接阈值 |
| `opencv_mouth_x`       | 嘴巴 X 坐标        | 无直接阈值 |
| `opencv_mouth_y`       | 嘴巴 Y 坐标        | 无直接阈值 |
| `opencv_nose_x`        | 鼻子 X 坐标        | 无直接阈值 |
| `opencv_nose_y`        | 鼻子 Y 坐标        | 无直接阈值 |
| `background_deviation` | 背景偏差值         | 越低越好   |
| `background_grayness`  | 背景灰度值         | 越低越好   |
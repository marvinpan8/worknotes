# BIQT 虹膜图像质量评估指标速查表

官网 https://github.com/mitre/biqt-iris

## 一、硬性指标 ⭐

| Column                            | 描述 (Description)                         | 建议阈值 / 合格标准     |
| --------------------------------- | ------------------------------------------ | ----------------------- |
| `quality`                         | 综合质量分数                               | **≥ 80**（越高越好）    |
| `iso_overall_quality`             | ISO 综合质量分数                           | **≥ 90**（越高越好）    |
| `iso_usable_iris_area`            | 可用虹膜面积占比（未被眼睑/睫毛/反光遮挡） | **≥ 80%**（建议 ≥ 90%） |
| `iso_margin_adequacy`             | 虹膜在图像中的居中度                       | **≥ 80**                |
| `iso_iris_pupil_concentricity`    | 虹膜与瞳孔中心重合度                       | **≥ 90**                |
| `iris_diameter`                   | 虹膜直径（像素）                           | **≥ 160**（建议 ≥ 200） |
| `pupil_circularity_avg_deviation` | 瞳孔圆形平均偏差                           | **≤ 0.5**（越低越圆）   |

## 二、建议指标

| Column                      | 描述 (Description)            | 建议阈值 / 合格标准           |
| --------------------------- | ----------------------------- | ----------------------------- |
| `contrast`                  | 整体图像对比度                | **≥ 50**                      |
| `sharpness`                 | 图像清晰度（原始值）          | **≥ 80**                      |
| `iso_sharpness`             | ISO 清晰度分数                | **≥ 8.0**                     |
| `iso_iris_pupil_contrast`   | 虹膜-瞳孔边界对比度           | **≥ 30**（ISO 推荐 ≥ 30）     |
| `iso_iris_sclera_contrast`  | 虹膜-巩膜边界对比度           | **≥ 5**（ISO 推荐 > 5）       |
| `iso_iris_pupil_ratio`      | 瞳孔缩放程度（虹膜/瞳孔比例） | **20 ~ 70**（ISO 推荐 20-70） |
| `iso_greyscale_utilization` | 灰度利用率（灰度值分布跨度）  | **≥ 6**（ISO 推荐 ≥ 6）       |
| `iris_pupil_gs`                   | 虹膜-瞳孔边界区分度   | **≥ 25**   |
| `iris_sclera_gs`                  | 虹膜-巩膜边界区分度   | **≥ 50**   |
| `pupil_circularity_avg_deviation` | 瞳孔圆形平均偏差      | **≤ 0.3**  |
| `normalized_*`                    | 各项归一化指标（0-1） | **≥ 0.80** |

## 三、辅助指标

| Column           | 描述 (Description)      | 建议阈值 / 合格标准 |
| ---------------- | ----------------------- | ------------------- |
| `image_width`    | 输入图像宽度（像素）    | ≥ 400               |
| `image_height`   | 输入图像高度（像素）    | ≥ 400               |
| `iris_center_x`  | 虹膜中心 X 坐标（像素） | 用于计算居中        |
| `iris_center_y`  | 虹膜中心 Y 坐标（像素） | 用于计算居中        |
| `pupil_center_x` | 瞳孔中心 X 坐标（像素） | 用于计算同心度      |
| `pupil_center_y` | 瞳孔中心 Y 坐标（像素） | 用于计算同心度      |
| `pupil_diameter` | 瞳孔直径（像素）        | 用于计算虹膜/瞳孔比 |
| `pupil_radius`   | 瞳孔半径（像素）        | 用于计算            |
| `normalized_contrast`                     | 归一化对比度              | ≥ 0.80 |
| `normalized_iris_diameter`                | 归一化虹膜直径            | ≥ 0.80 |
| `normalized_iris_pupil_gs`                | 归一化虹膜-瞳孔边界区分度 | ≥ 0.80 |
| `normalized_iris_sclera_gs`               | 归一化虹膜-巩膜边界区分度 | ≥ 0.80 |
| `normalized_iso_iris_pupil_concentricity` | 归一化虹膜-瞳孔同心度     | ≥ 0.80 |
| `normalized_iso_iris_pupil_contrast`      | 归一化虹膜-瞳孔对比度     | ≥ 0.80 |
| `normalized_iso_sharpness`                | 归一化清晰度              | ≥ 0.80 |
| `log`                                     | 处理日志                  | 仅参考 |
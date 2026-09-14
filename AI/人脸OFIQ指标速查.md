# OFIQ 人脸图像质量量化评估速查表

OFIQ官网：https://bsi.bund.de/dok/OFIQ-e

GItHub：https://github.com/BSI-OFIQ/OFIQ-Project

参考标准: 

- **ISO/IEC 19794-5:2011/2005  数据交换格式标准**  https://www.correlance.com/cms/en/iso19794-5
- **ISO/IEC 29794-5:2025  人脸图像质量评估标准**  https://www.iso.org/standard/81005.html
- **ISO/IEC 39794-5:2019  人脸图像数据的可扩展交换格式**
-  **ISO/IEC 19785-1:2020  通用生物特征交换格式框架（CBEFF）**
- **ANSI INCITS 398-2008/2005   美国标准，通用生物特征交换格式框架 (CBEFF)**

数据集：https://miatbiolab.csr.unibo.it/icao-synthetic-dataset/

- **African (EAF)**            European / American (EEA)      Indian / Asian (EIA)
- East-Asian (EAS)        Middle Eastern (EME)--中东

## 一、硬性指标 ⭐

| Column    | 描述 (Description)    | 建议阈值 / 合格标准  |
| ---------- | ---------- | --------------- |
| `unified_quality_score` | 综合质量原始分[15, 30] | 23=50分，越高越好 |
| `unified_quality_score_scalar` | **综合质量分数** | **≥ 70**（严格场景 ≥ 80）  |
| `sharpness`    | 清晰度原始分 | -20=0分，-14=55分 |
| `sharpness_scalar`  | 清晰度标化分数（0-100） | **≥ 50**（严格场景 ≥ 65）|
| `head_size`  | 头部占图像高度的比例   | 0.45=100 0.38=40  0.50=54  0.31=12 |
| `head_size_scalar` | 头部占比与0.45的偏差分 | **≥ 40**  |
| `head_pose_yaw_scalar`   | 偏航角标化分数（算法未知） | -12=95 |
| `head_pose_pitch_scalar` | 俯仰角标化分数（算法未知） | 11=96 |
| `head_pose_roll_scalar`  | 翻滚角标化分数（算法未知） | -12=95                              |
| `head_pose_yaw`  | 偏航角原始值（度）      | 绝对值 ≤ 5° (ICAO标准) |
| `head_pose_pitch` | 俯仰角原始值（度）      | 绝对值 ≤ 5°(ICAO标准) |
| `head_pose_roll`   | 翻滚角原始值（度）      | 绝对值 ≤ 8°(ICAO标准，最好5°) |
| `inter_eye_distance` | **瞳孔间距IPD（像素**） | **≥ 90**（ICAO 芯片要求90 最好120） |
| `inter_eye_distance_scalar` | 瞳孔间距与70像素的偏差分 | **≥ 80**  |
| `eyes_visible`               | 眼睛可见区域（EVZ）原始值 | 越高越好 |
| `eyes_visible_scalar`  | 眼睛可见区域（EVZ）标化分数 | **≥ 80**  |
| `eyes_open`                  | 双眼睁开原始值   | 闭合0.01~0.07睁开 |
| `eyes_open_scalar`             | 双眼睁开标化分数 | **≥ 80** |
| `mouth_closed`               | 嘴巴闭合原始值   | 闭合0.01~0.4张开 |
| `mouth_closed_scalar`          | 嘴巴闭合标化分数 | **≥ 80** |
| `mouth_occlusion_prevention` | 嘴巴无遮挡原始值 | 越高越好 |
| `mouth_occlusion_prevention_scalar` | 嘴巴无遮挡标化分数 | **≥ 80**  |
| `face_occlusion_prevention`  | 面部无遮挡原始值 | 越高越好 |
| `face_occlusion_prevention_scalar`  | 面部无遮挡标化分数 | **≥ 80** |
| `no_head_coverings`                     | 无头饰原始值       | 0无遮挡，1全遮挡 |
| `no_head_coverings_scalar`  | 无头饰标化分数 | **≥ 80**  |
| `expression_neutrality`      | 表情中性原始值     | -5千=50分 0=73分 1万=90分 |
| `expression_neutrality_scalar` | 表情中性标化分数 | **≥ 70** |
| `single_face_present`        | 单张人脸原始值   | 0 |
| `single_face_present_scalar`   | 单（多）张人脸标化分数（0-100） | **100** |

## 二、拍摄设备相关

| Column                             | 描述 (Description)          | 建议阈值 / 合格标准            |
| ---------------------------------- | --------------------------- | ------------------------------ |
| `illumination_uniformity`   | 光照均匀度原始值 | 0.32=71  0.72=91    |
| `illumination_uniformity_scalar`   | 光照均匀度标化分数（0-100） | **≥ 80** |
| `background_uniformity`     | 背景均匀度原始值 | 9=95 65=69 |
| `background_uniformity_scalar` | 背景均匀度标化分数（0-100） | **≥ 80** |
| `luminance_mean`        | 亮度均值原始值         | 0.01=3 0.4=97 |
| `luminance_mean_scalar` | 亮度均值标化分数   | **≥ 80** |
| `luminance_variance`    | 亮度方差原始值         | 0=3  0.02=94 |
| `luminance_variance_scalar` | 亮度方差标化分数（0-100） | **≥ 80** |
| `under_exposure_prevention` | 防欠曝原始值     | 0 |
| `under_exposure_prevention_scalar` | 防欠曝标化分数（0-100） | **≥ 80** |
| `over_exposure_prevention`  | 防过曝原始值     | 0 |
| `over_exposure_prevention_scalar`  | 防过曝标化分数（0-100）     | **≥ 80**  |
| `dynamic_range`             | 动态范围原始值   | 2=33 5=63  7=93 |
| `dynamic_range_scalar`  | 动态范围标化分数（0-100）   | **≥ 80**  |
| `compression_artifacts`     | 压缩伪影原始值   | 0 |
| `compression_artifacts_scalar` | 压缩伪影标化分数（0-100）   | **≥ 80** |
| `natural_colour`             | 颜色自然度原始值 | 0                   |
| `natural_colour_scalar`        | 颜色自然度标化分数（0-100） | **≥ 80** |

## 三、辅助指标

| Column                  | 描述 (Description)     | 建议阈值 / 合格标准  |
| ----------------------- | ------------------| -------------------- |
| `margin_above_of_the_face_image`   | 头顶边距原始值 | 1.38=46 |
| `margin_below_of_the_face_image`   | 下巴边距原始值 | 1.69=25 |
| `leftward_crop_of_the_face_image`  | 左侧裁切原始值 | 2.56=100 |
| `rightward_crop_of_the_face_image` | 右侧裁切原始值 | 2.62=100            |
| `margin_above_of_the_face_image_scalar`   | 头顶边距标化分数   | ≥ 80  |
| `margin_below_of_the_face_image_scalar`   | 下巴边距标化分数   | ≥ 80  |
| `leftward_crop_of_the_face_image_scalar`  | 左侧未裁切标化分数 | ≥ 80  |
| `rightward_crop_of_the_face_image_scalar` | 右侧未裁切标化分数 | ≥ 80  |











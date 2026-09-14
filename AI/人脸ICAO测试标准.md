# ICAO 人脸测试标准

- https://www.icao.int/sites/default/files/TRIP/Publications/TR-Portrait-Quality-v1.0.pdf

- https://nigos.nist.gov/ifpc2018/presentations/41_wolf_NIST_IFPC_2018_Andreas_Wolf_20181123.pdf

  

  **页33表8 - 姿态角度要求与最佳实践 (Table 8 - Pose angle requirements and recommendations)**

| 标准 (Criterion)           | 要求 (Requirement)                                           | 最佳实践 (Best Practice)                                     |
| -------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 姿态角度 (Pose angle)-33页 | 俯仰角 (Pitch) ≤ ±5°, 偏航角 (Yaw) ≤ ±5°, 滚动角 (Roll) ≤ ±8° | 俯仰角 (Pitch) ≤ ±5°, 偏航角 (Yaw) ≤ ±5°, 滚动角 (Roll) ≤ ±5° |
| IED-52页                   | IED ≥ 90 像素                                                | IED ≥ 120 像素                                               |

​     **页40表9 - 人像几何要求 (Table 9 - Geometric portrait requirements)**

| 术语 (Term) | 描述 (Description)                | 要求 (Requirement) |
| ----------- | --------------------------------- | ------------------ |
| **Mh**      | 从图像左侧到面部中心点M的水平距离 | 45% ≤ Mh/A ≤ 55%   |
| **Mv**      | 从图像顶部到面部中心点M的垂直距离 | 30% ≤ Mv/B ≤ 50%   |
| **W/A**     | 头部宽度与图像宽度的比值          | 50% ≤ W/A ≤ 75%    |
| **L/B**     | 头部长度与图像高度的比值          | 60% ≤ L/B ≤ 90%    |

- **表情**：面部应为**中性表情**，**不得微笑**，嘴巴应**闭合**（牙齿不可见），眉毛不得上扬。
- **眼睛**：双眼应**自然睁开**，瞳孔和虹膜（包括颜色）应**完全可见**。眼睛应注视相机。两眼之间的距离应略小于画面宽度的**1/4**。
- **压缩**：**JPEG 压缩比不得超过 15:1**。
- **最小尺寸**：捕获的人像（裁剪后）的最小尺寸应为 **1200像素（宽）× 1600像素（高）**。
- **背景**：背景表面应**素净**，无可见斑点、线条或曲线纹理，颜色应**均匀**。背景中不应出现任何物体（如椅子、家具）
- **照明均匀性**：面部照明应**均匀分布**，尤其要**对称**（左右脸亮度无差异）。在四个测量区域（额头、左右脸颊、下巴）中，**任意通道的最低平均强度值不应低于最高值的50%**。







### TONO测试ICAO要求

- https://miatbiolab.csr.unibo.it/icao-synthetic-dataset/

| No. | Description of the test (英文) | 测试描述 (中文) |
| --- | ------ | ------ |
|  | **Basic checks** | **基础检查** |
| 1 | Unique and valid face | 存在唯一且有效的人脸 |
| 2 | Face fully included in image frame | 人脸完全包含在图像框内 |
|  | **Geometric tests** | **几何测试** |
| 3 | Eye distance | 双眼间距 |
| 4 | Horizontal/vertical position | 水平/垂直位置 |
| 5 | Head image width/height ratio | 头部图像的宽高比 |
|  | **Photographic tests** | **摄影/图像质量测试** |
| 6 | Face is correctly focused | 人脸对焦正确 |
| 7 | Sharpness of the image | 图像清晰度 |
| 8 | Face saturation | 人脸色彩饱和度 |
| 9 | Image color conformance | 图像色彩符合性 |
| 10 | Shadows over the face | 面部存在阴影 |
| 11 | Glasses with dark colored lenses or glare | 佩戴深色镜片或镜片反光的眼镜 |
| 12 | Cluttered background | 背景杂乱 |
|  | **Pose and facial attributes tests** | **姿态与面部属性测试** |
| 13 | Gaze direction | 视线方向 |
| 14 | Mouth expression | 嘴部表情 |
| 15 | Correct position of shoulders | 肩部位置是否正确 |
| 16 | Both eyes visible and open | 双眼可见且睁开 |
| 17 | Eyes color | 眼睛颜色 |
| 18 | Eyes occluded by glasses or hair | 眼睛被眼镜或头发遮挡 |
| 19 | Presence of glasses | 是否佩戴眼镜 |
| 20 | Glasses' frames too heavy | 眼镜框过粗/过重 |
| 21 | Presence of hat/cap on head | 头部佩戴帽子/头饰 |
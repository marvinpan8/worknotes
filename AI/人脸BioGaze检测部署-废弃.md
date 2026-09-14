# ubuntu 部署 BioGaze 人脸检测模型

官网：https://github.com/Maphoz/BioGaze

```properties
mkdir /data/python/bioGaze
# 安装 python隔离环境 并激活
python3.10 -m venv .venv
source /data/python/BioGaze/.venv/bin/activate


git clone https://github.com/Maphoz/BioGaze.git
cd BioGaze
pip install -r requirements.txt

# 升级基础包
pip install --upgrade pip setuptools wheel
pip install ultralytics==8.2.85
pip install numpy==1.26.2
pip install opencv-contrib-python==4.8.0.76


sudo apt install build-essential cmake pkg-config
sudo apt install libx11-dev libatlas-base-dev
sudo apt install libgtk-3-dev libboost-python-dev
sudo apt install python3-dev
# 卸载所有包
# pip freeze | xargs pip uninstall -y
# deactivate
mkdir input output
```

### 下载模型文件

- github 官网要求

```properties
shape_predictor_68_face_landmarks.dat
epoch_24_ckpt.pth.tar
79999_iter.pth
```

- 代码要求下载 https://github.com/akanametov/yolo-face/releases/tag/1.0.0/yolov8n-face.pt

```properties
vim detectors/detect.py
# 修改地址
model_path = "/data/python/BioGaze/BioGaze/yolo/yolov8n-face.pt"
```

## 执行

```properties
# 单项检查
python specific_checks.py -i input/1.jpg -c all
# 综合分析
python quality_analysis.py -i input/TONO_mixed -o output
```


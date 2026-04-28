# 项目整体架构与运行思路

## 概述

这是一个 **5 步流水线式的游戏目标检测+自动化项目**，使用 YOLOv4-tiny 在 Windows 游戏窗口上进行实时物体检测和鼠标自动化操作。

## 文件树

```
├── 1_generate_dataset.ipynb       # 步骤1：截图数据集生成
├── 2_label_dataset.ipynb          # 步骤2：数据集标注与配置准备
├── 3_yolo_model_training.ipynb    # 步骤3：模型训练（Google Colab）
├── 4_yolo_opencv_detector.ipynb   # 步骤4：OpenCV 实时检测
├── 5_automate_actions.ipynb       # 步骤5：游戏动作自动化（pynput）
├── README.md                      # 项目文档与教程
├── requirements.txt               # Python 依赖
├── yolov4-tiny/                   # YOLOv4-tiny 模型工件
│   ├── obj.data                   # Darknet 数据配置（训练/验证路径）
│   ├── obj.names                  # 类别标签
│   ├── process.py                 # 训练/测试集划分脚本
│   ├── yolov4-tiny.conv.29        # 预训练卷积权重（二进制）
│   ├── yolov4-tiny-custom.cfg     # 最终 Darknet 模型架构
│   ├── yolov4-tiny-custom_template.cfg  # 模板配置（含占位符）
│   └── training/
│       └── placeholder.txt        # 占位文件
└── docs/
    └── PROJECT_ARCHITECTURE.md    # 本文档
```

## 技术栈

| 维度 | 详情 |
|------|------|
| **语言** | Python 3.11+ |
| **ML 框架** | Darknet (AlexeyAB 分支) — YOLOv4-tiny 训练 |
| **推理引擎** | OpenCV DNN 模块 (`cv.dnn`) — 加载 Darknet 模型进行 CPU 推理 |
| **计算机视觉** | OpenCV (`opencv-python`) |
| **Windows GUI** | pywin32 (`win32gui`, `win32ui`, `win32con`) — Win32 API 绑定 |
| **自动化** | pynput — 鼠标/键盘控制 |
| **图像处理** | Pillow, numpy |
| **环境** | Jupyter Notebook（本地）+ Google Colab（GPU 训练） |

## 整体流程

```
数据集生成      标注与配置         模型训练          实时检测          自动化操作
(Notebook 1)   (Notebook 2)     (Notebook 3)     (Notebook 4)     (Notebook 5)

WindowCapture  LabelUtils       Google Colab     WindowCapture    WindowCapture
(Win32 API)    shuffle + zip    + Darknet        ImageProcessor   ImageProcessor
     |              |                |                |                |
     v              v                v                v                v
images/*.jpg → shuffled_images/  → Darknet Train  → OpenCV DNN    → pynput
                 obj.zip → Drive   yolov4-tiny-     Inference Loop   Mouse Actions
                 cfg/obj.data/     custom_last.     (live window     (click, drag)
                 obj.names         .weights         capture)
```

## 5 个步骤详解

### 步骤 1: 截图采集 — `1_generate_dataset.ipynb`

**核心类：`WindowCapture`**

使用 Win32 API 实时截取指定游戏窗口：

- `win32gui.FindWindow()` — 按标题查找窗口
- `BitBlt` (GDI) — 截取客户区图像
- 自动去除窗口边框（8px 边框，30px 标题栏）
- 返回 NumPy RGB 数组（H × W × 3）
- `generate_image_dataset()` — 持续保存截图为 `images/img_xxx.jpg`

启动后进入无限截图循环（0.3s 间隔），将训练数据保存到磁盘。

---

### 步骤 2: 数据标注与配置生成 — `2_label_dataset.ipynb`

**核心类：`LabelUtils`**

| 方法 | 功能 |
|------|------|
| `create_shuffled_images_folder()` | 随机打乱 `images/*.jpg` 到 `shuffled_images/`，防止标注偏差 |
| `create_labeled_images_zip_file()` | 将标注后的 `.txt` + `.jpg` 移入 `obj/` 并打包为 `yolov4-tiny/obj.zip` |
| `update_config_files(classes)` | 用实际值替换模板中的占位符，生成最终 `.cfg` 和 `obj.names` |

**模板替换公式：**

| 占位符 | 替换规则 |
|--------|----------|
| `_CLASS_NUMBER_` | 实际类别数 |
| `_NUMBER_OF_FILTERS_` | `(num_classes + 5) × 3`（Darknet 公式） |
| `_MAX_BATCHES_` | `max(6000, num_classes × 2000)` |

标注流程：打乱 → 上传 [makesense.ai](https://www.makesense.ai/) 人工标注 → 下载 → 打包 → 配置生成。

**辅助脚本 `yolov4-tiny/process.py`：** 对 `obj/` 目录按 90/10 比例生成 `train.txt` / `test.txt` 划分。

---

### 步骤 3: 模型训练 — `3_yolo_model_training.ipynb`

**运行环境：Google Colab（GPU 加速）**

训练流程：

1. 克隆 `AlexeyAB/darknet` 仓库
2. 挂载 Google Drive（从 `yolov4-tiny/` 加载自定义文件）
3. 在 Makefile 中启用 GPU/CUDNN/OpenCV 并编译
4. 清空默认 data/cfg，复制自定义文件
5. 运行 `process.py` 生成 train/test 划分
6. 启动训练：
   ```
   ./darknet detector train data/obj.data cfg/yolov4-tiny-custom.cfg yolov4-tiny.conv.29 -dont_show
   ```
7. 保存权重到 Google Drive: `training/yolov4-tiny-custom_last.weights`

**模型规格：**

| 参数 | 值 |
|------|-----|
| 架构 | YOLOv4-tiny（轻量级） |
| 输入尺寸 | 416 × 416 RGB |
| 检测头 | 2 个（13×13 + 26×26） |
| 锚点框 | 6 个默认锚点 |
| 类别数 | 3（assaulter, freezer, heater） |
| 训练超参 | batch=64, subdivisions=16, lr=0.00261, max_batches=6000 |

采用**迁移学习**：backbone 使用 COCO/ImageNet 预训练的 `yolov4-tiny.conv.29`（29 层卷积骨干）权重进行初始化，而非随机初始化；YOLO 检测头从零开始随机初始化。所有层在训练过程中共同进行全模型微调（cfg 中未设置 `stopbackward=1`，未冻结任何层）。

---

### 步骤 4: 实时检测 — `4_yolo_opencv_detector.ipynb`（核心模块）

**核心类：`ImageProcessor`**

这是项目的主推理引擎，流程如下：

```
截图 → 转blob(416×416, 1/255) → net.forward() → 解析输出 → NMS → 绘制框 → 返回坐标
```

**初始化 (`__init__`)：**
- 通过 `cv.dnn.readNetFromDarknet(cfg, weights)` 加载模型
- 设置后端 `DNN_BACKEND_OPENCV`（纯 CPU 推理）
- 获取输出层名称（`getUnconnectedOutLayers`）
- 从 `obj.names` 读取类别名称

**推理流程 (`proccess_image`)：**

1. `cv.dnn.blobFromImage(img, 1/255, (416,416))` — 图像预处理
2. `net.forward(output_layers)` — 前向传播
3. `np.vstack()` — 堆叠所有检测头输出
4. `get_coordinates()` — 解析坐标 + NMS
5. `draw_identified_objects()` — 绘制边界框

**坐标解析 (`get_coordinates`)：**

- 每个检测输出：`[cx, cy, w, h, confidence, class_scores...]`
- 反归一化到屏幕像素坐标
- `cv.dnn.NMSBoxes()` 进行非极大值抑制（阈值 = confidence - 0.1）
- 返回列表：`{x, y, w, h, class, class_name}`

**运行方式：** 无限循环：捕获 → 检测 → 绘制 → 打印坐标 → 按 `q` 退出。

---

### 步骤 5: 自动化操作 — `5_automate_actions.ipynb`

在检测基础上，用 `pynput` 控制鼠标，包含 3 种策略：

| 策略 | 说明 |
|------|------|
| **简单点击** | 过滤目标类别 → 点击第一个检测到的物体 |
| **速度预测滑动** | 两帧差分计算物体速度 → `velocity_multiplyer=2` 预测未来位置 → 鼠标拖拽 |
| **随机点击** | 200 次随机位置点击 |

## 设计模式与架构特点

1. **流水线模式（Pipeline）：** 5 个 Notebook 严格顺序执行：生成 → 标注 → 训练 → 检测 → 自动化。

2. **自包含 Notebook：** 每个 Notebook 独立复制 `WindowCapture` 类（而非模块导入），确保各自可独立运行。这是教程式项目的典型设计。

3. **模板方法模式：** `LabelUtils.update_config_files()` 用字符串占位符实现配置代码生成。

4. **迁移学习：** 训练阶段 backbone 用 `yolov4-tiny.conv.29` 预训练权重初始化（非随机），检测头随机初始化；所有层共同进行全模型微调，未冻结任何层。

5. **两帧速度预测：** Notebook 5 采用帧差法计算物体速度，预测未来位置——一种 ad-hoc 运动预测模式。

6. **资源管理：** Win32 截图代码中细致管理 GDI 资源（`DeleteDC`、`ReleaseDC`、`DeleteObject`），防止 Windows 内存泄漏。

## 入口点

| Notebook | 入口点 | 功能 |
|----------|--------|------|
| `1_generate_dataset.ipynb` | Cell 3: 无限截图循环 | 采集训练数据 |
| `2_label_dataset.ipynb` | 多个 `LabelUtils` 方法调用 | 标注与配置生成 |
| `3_yolo_model_training.ipynb` | `./darknet detector train ...` | GPU 模型训练 |
| `4_yolo_opencv_detector.ipynb` | `while(True): improc.proccess_image(wincap.get_screenshot())` | **核心检测循环** |
| `5_automate_actions.ipynb` | 多种 `while(True)` 循环 | 鼠标自动化 |

终端用户通常直接运行 Notebook 4（检测）或 Notebook 5（自动化），前提是已完成模型训练。

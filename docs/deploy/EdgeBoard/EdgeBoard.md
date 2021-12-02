# EdgeBoard部署

## 简介

EdgeBoard集成了PaddleLite预测库和预测的demo，PaddleX模型可直接在EdgeBoard上进行预测部署。

## 环境依赖

硬件支持型号：EdgeBoard FZ3/FZ5/FZ9系列

操作系统：Ubuntu系统

软核版本：1.8.1

模型训练环境：PaddlePaddle <=2.1.2、PaddleX = 1.3.0

## 环境安装

EdgeBoard在1.8.1版本后，出厂自带ubuntu操作系统和对应的预测环境，如果您手中的设备不是1.8.1版本，可以通过官方文档自行升级系统和软核：https://ai.baidu.com/ai-doc/HWCE/Fkuqounlk

**注意：**EdgeBoard系统内部已经包含了PaddleLite预测库，无需再次安装。

## 模型导出

EdgeBoard支持Paddle inference格式的模型，在EdgeBoard上部署模型时需要将训练过程中保存的模型导出为inference格式模型，导出的inference格式模型包括`__model__`、`__params__`、`model.yml`三个文件；检查使用PaddleX训练好的模型文件，一般存在与best_model文件夹下，以yolov3_darknet53为例，转换命令如下：

```text
paddlex --export_inference --model_dir=/home/aistudio/output/yolov3_darknet53/best_model --save_dir=./inference_model --fixed_input_shape=[416,416]
```

| 参数                | 说明                                                  |
| ------------------- | ----------------------------------------------------- |
| --export_inference  | 是否将模型导出为用于部署的inference格式，指定即为True |
| --model_dir         | 待导出的模型路径                                      |
| --save_dir          | 导出的模型存储路径                                    |
| --fixed_input_shape | 固定导出模型的输入大小，默认值为None                  |

使用TensorRT预测时，需固定模型的输入大小，通过`--fixed_input_shape`来制定输入大小[w,h]。

**注意**：

- 分类模型的固定输入大小请保持与训练时的输入大小一致；
- 检测模型模型中YOLO系列请保存w与h一致，且为32的倍数大小；RCNN类无此限制，按需设定即可
- 指定[w,h]时，w和h中间逗号隔开，不允许存在空格等其他字符。
- 需要注意的，w,h设得越大，模型在预测过程中所需要的耗时和内存/显存占用越高；设得太小，会影响模型精度

更多模型转换详情参考PaddleX官方文档：https://paddlex.readthedocs.io/zh_CN/develop/deploy/export_model.html

## 模型部署

**PaddleLiteDemo包含C++示例和Python示例，以下说明基于Edgeboard 1.8.1版本的示例进行介绍**

### Step1:上传模型

模型训练完成后，将模型文件上传到Edgeboard内，存放于`/root/workspace/PaddleLiteDemo/res/models`选择与模型相对应的文件夹，比如yolov3模型存放在detection目录,模型文件中需要包含于示例对应的文件，包括

`model`, `params`：模型文件名称

`label_list.txt`：模型标签文件

`img`：放置预测图片的文件夹

`config.json`：配置文件，需要基于示例给出的结构，更改为自训练模型的参数

可以通过scp方式上传模型，详情参考【EdgeBoard文件传输方式】：https://ai.baidu.com/ai-doc/HWCE/8kqg9jy7x

### Step2:修改模型配置文件参数

模型配置文件config.json存放于`/root/workspace/PaddleLiteDemo/res/models`

config.json文件内容如下：

```json
{
        "network_type":"YOLOV3",

        "model_file_name":"model",
        "params_file_name":"params",

        "labels_file_name":"label_list.txt",

        "format":"RGB",
        "input_width":608,
        "input_height":608,

        "mean":[123.675, 116.28, 103.53],
        "scale":[0.0171248, 0.017507, 0.0174292],
        "threshold":0.3
}
```

**注意**：config.json文件内的mean、scale计算方式与训练模型时的计算方式可能不同：

比如R为一张图片中一个像素点的R通道的值，训练模型时，一般为（R / 255 – mean） / std；

EdgeBoard的PaddleLiteDemo中为提高计算效率，使用（R – mean‘) * scale’方式。

所以就需要通过如下运算，将训练中的均值和方差计算为Edgeboard的mean‘和scale’，计算方式如下：

```sh
   mean’ = mean * 255

   scale‘ = 1 / ( 255 * std )
```

### Step3:修改系统参数配置文件

系统配置文件存放于`/root/workspace/PaddleLiteDemo/configs`内，根据模型属性修改对应的配置文件，比如yolov3模型，修改image.json文件中模型文件路径以及预测图片路径。

```json
{
    "model_config": "../../res/models/detection/yolov3",
    "input": {
        "type": "image",
        "path": "../../res/models/detection/yolov3/img/screw.jpg"
    },
    "debug": {
        "display_enable": true,
        "predict_log_enable": true,
        "predict_time_log_enable": false
    }
}
```

**配置文件参数说明：**

`model_config` : 模型的目录,相对于 `PaddleLiteDemo/C++/build` 目录

`type`： 输入源的类型，可选项目，支持的value有：image、usb_camera、rtsp_camera、multichannel_camera

`path`: 输入源对应的 设备节点或url 或 图片的名字

| type                | path                                 | 备注               | 支持的设备 |
| ------------------- | ------------------------------------ | ------------------ | ---------- |
| image               | ../../res/models/XXX                 | 使用图片作为输入源 | ALL        |
| usb_camera          | /dev/video0                          | USB摄像头          | FZ3/FZ5    |
| usb_camera          | /dev/video2                          | USB摄像头          | FZ9D       |
| rtsp_camera         | rtsp://admin:eb123456@192.168.1.167/ | RTSP摄像头         | FZ5        |
| multichannel_camera | /dev/video1                          | 多路分时复用摄像头 | FZ9D       |

**注意：** 

`multichannel_camera`：仅FZ9D 支持该type

`display_enable`: 显示器显示开关，当没有显示器时可以设置为 false

`predict_log_enable` : 预测的结果通过命令行打印的开关

`predict_time_log_enable`: 模型预测单帧耗时的打印开关（不包括图像处理时间）

### Step4:程序编译

使用C++示例，需要进行程序编译，Python无需关注。

```sh
cd /root/workspace/PaddleLiteDemo/C++
mkdir build
cd build
cmake ..
make
```

### Step5:执行样例

#### 图像推理

模型配置文件和系统配置文件修改完成后，运行程序

C++示例

```sh
cd /root/workspace/PaddleLiteDemo/C++/build
./detection ../../configs/detection/yolov3/image.json
```

Python示例

```sh
cd /root/workspace/PaddleLite/Python/demo
python3 detection.py ../../configs/detection/yolov3/image.json
```

#### USB摄像头推理

C++示例

```sh
cd /root/workspace/PaddleLiteDemo/C++/build
./detection ../../configs/detection/yolov3/usb_camera.json
```

Python示例

```sh
cd /root/workspace/PaddleLite/Python/demo
python3 detection.py ../../configs/detection/yolov3/usb_camera.json
```

## PaddleX模型支持列表

| 类别 | PaddleX模型                 | 输入尺寸 | Edgeboard支持情况 | FZ9推理时间 |
| ---- | --------------------------- | -------- | ----------------- | ----------- |
| 分类 | MobilenetV1                 | 224x224  | 支持              | 7.36ms      |
| 分类 | MobilenetV2                 | 224x224  | 支持              | 10.56ms     |
| 分类 | Res18                       | 224x224  | 支持              | 6.58ms      |
| 分类 | Res34                       | 224x224  | 支持              | 11.28ms     |
| 分类 | Res50                       | 224x224  | 支持              | 17.58ms     |
| 分类 | Res50_vd                    | 224x224  | 支持              | 18.55ms     |
| 分类 | Res50_vd_ssld               | 224x224  | 支持              | 18.56ms     |
| 分类 | Res101                      | 224x224  | 支持              | 29.82ms     |
| 分类 | Res101_vd                   | 224x224  | 支持              | 30.78ms     |
| 分类 | Res101_vd_ssld              | 224x224  | 支持              | 30.79ms     |
| 分类 | DenseNet121                 | 224x224  | 支持              | 46.01ms     |
| 分类 | DenseNet161                 | 224x224  | 支持              | 90.29ms     |
| 分类 | DenseNet201                 | 224x224  | 支持              | 78.20ms     |
| 分类 | HRNet_W18                   | 224x224  | 支持              | 50.34ms     |
| 分类 | Xception41                  | 224x224  | 支持              | 68.18ms     |
| 分类 | Xception65                  | 224x224  | 支持              | 60.99ms     |
| 分类 | ShuffleNetV2                | 224x224  | 不支持            | -           |
| 分类 | AlexNet                     | 224x224  | 不支持            | -           |
| 分类 | MobilenetV3                 | 224x224  | 不支持            | -           |
| 检测 | YOLOV3-Darknet53            | 608x608  | 支持              | 129.29ms    |
| 检测 | YOLOV3-Res34                | 608x608  | 支持              | 79.58ms     |
| 检测 | YOLOV3-MobilenetV1          | 608x608  | 支持              | 77.12ms     |
| 检测 | YOLOV3-MobilenetV3_large    | 608x608  | 不支持            | -           |
| 检测 | FasterRCNN系列              | -        | 不支持            | -           |
| 检测 | PPYOLO                      | -        | 不支持            | -           |
| 分割 | deeplabv3P_MobileNetV2_x1.0 | -        | 支持              | -           |
| 分割 | MaskRCNN系列                | -        | 不支持            | -           |
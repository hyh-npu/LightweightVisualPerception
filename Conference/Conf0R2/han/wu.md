# 问题调研板块
---
## 验证数据集的选择
---
### 数据集相关信息
  1. EuROC MAV数据集
    
* 基本信息

  EuRoC MAV（EuRoC Micro Aerial Vehicle）由苏黎世联邦理工 ETH-ASL 实验室发布，是视觉惯性 VIO/SLAM 领域行业标准基准数据集
  专门用于微型无人机室内视觉惯性里程计、双目 / 单目 VIO、视觉 SLAM 算法评测。

  数据采集硬件主要为：左目相机（cam0）、右目相机（cam1）、IMU（imu0）、激光追踪器（leica0）

  数据集下载有两种下载格式，分别为zip格式与rosbag格式
  
  zip格式结构如下(以MH_01序列为例)

  ```
  MH_01/
    mav0/
    ├── body.yaml        # MAV机体基础定义，极少使用
    ├── cam0/            # 左目相机
    │   ├── data/        # 灰度PNG图像，文件名=纳秒时间戳
    │   ├── data.csv     # 两列：#timestamp [ns], filename
    │   └── sensor.yaml  # 相机内参、畸变、相机到IMU的外参T_bc
    ├── cam1/            # 右目相机，结构同cam0
    ├── imu0/
    │   ├── data.csv     # IMU原始测量数据
    │   └── sensor.yaml  # IMU噪声参数、采样频率、外参
    ├── leica0/          # 激光跟踪真值（部分序列有）
    └── state_groundtruth_estimate0/  # 官方推荐真值（Vicon融合）
        └── data.csv
  ```

  rosbag格式结构如下（即四个硬件设备所对应的话题）
  ```
  /cam0/image_raw     #左目灰度图像
  /cam1/image_raw     #右目灰度图像
  /imu0               #IMU原始数据
  /leica/position     #激光跟踪真值
  ```
  * 运行步骤

  1. 从 ETH ASL 官方页面下载目标序列；


  2. 按原始目录结构解压；


  3. 读取 data.csv 建立图像和 IMU 时间序列；


  4. 从 sensor.yaml 加载相机内参、畸变参数和外参；


  5. 将 IMU 角速度和加速度送入预积分模块；


  6. 将图像送入前端特征提取或直接法模块；


  7. 读取 state_groundtruth_estimate0/data.csv 作为评测真值；


  8. 通过 evo 或自有评测程序计算 ATE 和 RPE。

  对于rosbag格式，下载后使用`roscore `挂载ros核心后，再`roslaunch + 对应算法节点`启动算法，最后使用`rosbag play + 包名 + --rate `
    实现可控速率播放数据集。仿真结果可通过订阅节点查看与保存。

* 其他

  * 评估方面
  
      可使用evo工具进行评估。evo 是一款开源 Python 命令行工具，全称 Evaluation of Odometry and SLAM，
      专门用于视觉 SLAM/VIO/ 激光里程计轨迹精度量化评估、轨迹可视化、格式互转，是视觉 SLAM 论文通用标准评测工具，替代 TUM 官方老旧 Python 评估脚本。
      使用该方法评估可能设计轨迹格式转换

  * 时间同步

     可能存在要依靠 csv 时间戳匹配图像与 IMU 测量值的情况（rosbag自动同步）

 2. MADMAX数据集（摩洛哥沙漠类比火星环境）
 
 * 基本信息

    该数据集是由德国宇航中心（DLR）使用自研便携式行星漫游传感器单元 SUPER，在摩洛哥撒哈拉荒漠完成实地采集，
    公开为MADMAX（摩洛哥采集火星模拟探索数据集）。

    数据集包含 36 组独立导航实验，采集于 8 处地貌、光照差异极大的火星模拟场地；总轨迹长度 9.2 km，单条最长轨迹 1.5 km。
    
    传感器采集数据包含：灰度双目相机、彩色单目相机、垂直布局全景双目相机、IMU 惯性单元全部带纳秒级时间戳记录。
    
    真值系统采用双天线实时动态差分（RTK）GNSS 融合算法，输出机器人 6 自由度位姿真值，并附带定位不确定性指标。

    8处数据集采集场地及其地貌特征如下：

    1.Rissani（A/B/C）：砂石混合，远景高地标，纹理充足，漫游可通行

    2.Kess Kess D：平坦碎石，纹理密集，视野开阔

    3.Kess Kess E：山谷乱石，坡度陡峭，现有漫游车无法通行；

    4.Maadid F：平缓卵石平地，标准简单场景

    5.Maadid G：巨型复合岩石，远近地标丰富

    6.Maadid H：纯沙丘，大面积无纹理区域，视觉算法高难度场景

    数据集总里程 9.2 km，单条轨迹长度 90 m–500 m 不等，每条轨迹自带天然闭环；
    
    数据集目录结构如下图所示
    
    ![目录结构图](C:\Users\22947\Desktop\图片\目录结构图.png)

    主要文件分为以下几类:

    1.标定文件夹：全部相机内参、相机 - IMU 外参、坐标系变换 CSV

    2.IMU数据：csv 时序文件，列：纳秒时间戳、三轴角速度、三轴加速度

    3.图像压缩包：校正左双目、校正右双目、校正彩色、全景上下原始图

    4.深度图：FPGA 实时 SGM 视差深度图（左相机基准）

    5.GNSS原始数据：RINEX 观测文件、星历、基站定位记录

    6.真值文件夹
        gt_5DoF_GNSS.csv：纯 GNSS 五自由度真值与gt_6DoF_GNSS_IMU.csv：融合六自由度真值

    7.元数据yaml：GNSS 与设备系统时间偏移、基站精确经纬度、序列起止时间

    8.算法基准结果：ORB-SLAM2、VINS-Mono 原始轨迹、对齐后 TUM 格式轨迹


3.TUM VI数据集

* 基本信息
TUM VI 是德国慕尼黑工业大学发布的视觉惯性数据集，主要用于单目、双目、视觉惯性里程计和 SLAM 算法评测。

  相比 EuRoC，TUM VI 更强调：

  更长时间运行；更复杂的室内环境；
  走廊、房间和大尺度空间；运动过程中曝光变化；
  多种相机视场和数据采集条件。

* 数据格式

```
dataset-calibration/
dataset-room1_512_16/
├── mav0/
│   ├── cam0/
│   │   ├── data/
│   │   └── data.csv
│   ├── cam1/
│   ├── imu0/
│   │   └── data.csv
│   └── state_groundtruth_estimate0/
│       └── data.csv

```
* 使用步骤

1. 进入官方页面，下载数据集标定文件；
 
2. 根据计算资源选择 room、corridor 或 magistrale 序列；

3. 下载相应分辨率版本；

4. 读取相机和 IMU 的 CSV 索引；

5. 加载官方相机模型和畸变参数；

6. 将鱼眼图像按照算法需要进行原始投影或矫正；

7. 对齐图像和 IMU 时间戳；

8. 使用真值文件进行 ATE、RPE 和漂移率评测。

* 数据集分析

  优点：序列长度较长；相比 EuRoC 具有更丰富的室内运动；对曝光变化、快速转动和长时间漂移更有挑战；
        适合验证视觉惯性前端的鲁棒性；与 EuRoC 配合可以检验算法是否过度依赖短序列和固定场景。

  缺点：对初学者而言，标定和鱼眼模型处理较复杂；场景仍以室内为主；
       真实火星地表纹理和光照特征缺失；数据集的相机模型与常规针孔相机实现可能不完全兼容；不同序列的真值可用范围和格式需要分别检查。





4.KITTI 数据集

* 基本信息

  KITTI 是自动驾驶和移动机器人领域最常用的数据集之一，由德国卡尔斯鲁厄理工学院和丰田技术研究院等机构联合发布。
  常用部分包括：KITTI Odometry；KITTI Raw Data；KITTI GPS/IMU；KITTI Vision Benchmark。
  其中KITTI Odometry 主要为道路、城市、乡村和高速公路场景

  常用数据包括：PNG 图像；times.txt 时间文件；calib.txt 相机投影矩阵；poses/*.txt 轨迹真值；
               Raw Data 中的 OXTS GPS/IMU 数据；Velodyne 点云。

* 数据集分析

  优点：场景范围大；、室外道路和大尺度运动丰富；具有较长距离轨迹；适合评估累计漂移；
       适合验证实时性、低频图像和大范围场景下的定位性能；工具链、基线算法和评测脚本成熟。

  劣势：主要是车载场景；运动以平面前进为主，三维姿态变化不足；图像和 IMU 的使用方式与 EuRoC/TUM VI 不同；
        KITTI Odometry 的标准版本并不总是包含可直接使用的原始 IMU；道路纹理、建筑和车辆与火星表面差异明显。



5.利用已有公开火星数据自行制作数据集

* 调研信息

  1.无同步高频 IMU 原始数据

  好奇号、毅力号 IMU 仅用于机载导航闭环，原始 100Hz 三轴角速度 / 加速度不对外公开；只有后端融合后的离散位姿，无独立 IMU 时序 csv

  2.图像时序不连续、帧率极低、丢帧严重

  火星通信带宽极度受限，巡视器不会持续每秒输出图像；Navcam 导航相机拍摄间隔几秒～几十秒，不存在 20Hz 连续双目同步图像流，没有帧间连续运动序列，无法模拟漫游车连续行驶的视觉里程计工况

  3.真值质量远低于地面 RTK-GT

  全局定位依赖轨道卫星遥感匹配，误差米级；
只有离散关键帧位姿，无逐帧高密度轨迹，不能用 evo 批量计算 ATE/RPE 做定量评测；
无时间严格对齐的双目同步图像：左右 Navcam 不同时曝光、时间戳差极大，无法直接双目 SLAM 输入。

  4.缺少统一同步时间戳体系

  图像、姿态、温度等数据时间基准不统一，需要复杂 SPICE 内核工具校正火星星时，对齐成本极高

  5.无原生深度图

  巡视器不在线输出视差深度，只能靠离线立体匹配生成，沙丘、弱纹理区域会大面积失效、产生无效深度掩码。


---

### 数据集下载

可通过网站[Awesome SLAM Datasets](https://sites.google.com/view/awesome-slam-datasets/home) (需要梯子)下载对应数据集

该网站至今仍在更新和维护，汇总了 SLAM 领域中常用的数据集，并根据 SLAM 研究中的细分领域和数据集特性做了分类

其对应的GitHub链接为[[youngguncho/awesome-slam-datasets: A curated list of awesome datasets for SLAM](https://github.com/youngguncho/awesome-slam-datasets)]

MADMAX Datasets下载地址为[morocco2018 | robotic datasets](https://datasets.arches-projekt.de/morocco2018/)


---

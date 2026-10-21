赛题四：基于 FPGA 的实时图像边缘检测系统
========================================
平台：易灵思 Titanium Ti60F225I3（Ti60F225 开发板）
工具：Efinity 2026.1
摄像头：SC431HAI（MIPI CSI-2，接 J4）
显示：HDMI（1080p60，接显示器）


一、系统数据流
--------------
  SC431HAI (MIPI CSI-2, J4)
        │
        ▼
  csi_rx_controller          CSI-2 协议解析（4 lane）
        │
        ▼
  i2c_master_ctrl_top        上电通过 I2C 配置传感器（ROM 内固化序列）
        │
        ▼
  frame_buffer_V3 + ddr3_top 帧缓存（4 像素/时钟写入，2 像素/时钟读出）
        │
        ▼
  debayer_top_2to1           RAW Bayer → RGB（2 像素/时钟，48bit）
        │
        ▼
  edge_pipeline_2px          rgb888→gray → Sobel 3×3 → |Gx|+|Gy| → 阈值二值化
        │
        ▼
  分屏合成器（顶层内）         左半 = 原始灰度全图(横向2:1)，右半 = 边缘全图(横向2:1)
        │                    右半输出为红/黑配色
        ▼
  dvi_encoder → HDMI 输出 1080p60


二、文件结构
------------
  ti60f225_oob.xml           Efinity 工程文件（65 个设计文件）
  ti60f225_oob.peri.xml      引脚/外设约束（Interface Designer）
  ti60f225_oob.sdc           时序约束
  debug_profile.wizard.json  调试核配置（ILA/VIO）
  debug_top.v                调试核 RTL（ILA/VIO 探针）
  Panel_1080p_reg.mem        MIPI 屏初始化码（本工程未实例化 DSI，不参与综合）
  rtl/                       Verilog 源码
    ti60f225_oob_top.v       顶层
    edge/                    边缘检测算法（rgb888_to_gray / sobel_3x3 / edge_pipeline_2px）
    key_autorepeat.v         按键消抖 + 长按连发
    cam/, cam_i2c_ctrl/      摄像头相关
    debayer/                 RAW→RGB
    frame_buffer_V3/         帧缓存
    ddr3_controller/         DDR3 控制器
    dvi_tx/                  HDMI 发送编码
    csi_rx_controller.sv     CSI-2 RX IP
  ip/                        IP 配置文件


三、编译与烧录
--------------
  编译（工作目录为本文件夹）：
      D:\enilisi\efinity\2026.1\bin\efx_run.bat ti60f225_oob.xml -f compile

  烧录到 SRAM（掉电失效，调试用）：
      efx_run.bat ti60f225_oob.xml -f program --pgm_opts mode=jtag

  烧录到 Flash（掉电保持，演示用）：
      用 Efinity Programmer 以 SPI Active 模式烧 outflow\ti60f225_oob.hex

  注意：本工程在 <efx:debugger> 中设置了 auto_instantiation=off，
        因为工程自带 debug_top.v（定义 debug_edb_top），不能由工具自动生成。


四、按键功能
------------
  KEY0（GPIOL_07）：边缘阈值 +32
  KEY1（GPIOL_03）：边缘阈值 -32
  按住约 0.4 秒后每 80ms 自动走一步（长按连发）
  阈值范围 32 ~ 4032，默认 128
  阈值越大 → 边缘越少、越干净；阈值越小 → 边缘越多


五、已实现的功能（对照赛题）
----------------------------
  基础要求：
    [√] 摄像头采集 640×480 以上（实际 1920×1080），行缓存齐全
    [√] 彩色转灰度，用常数乘法（综合为移位加法），无除法器
    [√] 3×3 Sobel，Gx/Gy 与题目给定卷积核一致
    [√] 梯度幅值 G = |Gx| + |Gy|（无开方）
    [√] 阈值二值化，阈值可参数化
    [√] HDMI 输出 1080p60（高于 640×480@60 要求）
    [√] 分屏显示（左灰度 / 右边缘），红/黑配色
  高阶挑战：
    [√] 实时阈值调节（按键 + 长按连发 + 阈值下限保护）
    [ ] 3×3 中值滤波（待加）
    [ ] 边缘叠加到原彩色图（机制已具备，待加模式切换）
    [ ] Canny / DDR 帧冻结回放（待加）


六、已知问题与后续
------------------
  1. 分屏为横向 2:1 压缩，画面宽度被压缩一半（正方形会显示为竖长条）。
     若要"全貌且长宽比正确"，需增加纵向 2:1（改 DDR 帧缓存读侧行步进），
     或改为"同一区域 1:1"分屏（只显示画面左半边，但长宽比正确、屏幕铺满）。
  2. 灰度流与边缘流之间存在约 1 行的垂直错位（edge_pipeline_2px 内部有 1 行延迟）。
     静态场景不可见，快速上下运动时可能出现一行差。
  3. 分屏合成器的行缓存目前综合到逻辑资源（LUT/FF），未进入 Block RAM，
     占用约 +8k LUT / +14k FF。改用同步读可让工具推断为 BRAM，节省约 16k LUT。
  4. Panel_1080p_reg.mem 与 MIPI DSI 通路保留但未使用（本工程输出走 HDMI）。


七、版本
--------
  2026-10-18  从厂家 Ti60f225_sc431hai2hdmi_v6 工程派生，加入 Sobel 边缘检测、
              按键调阈、灰度/边缘分屏合成器、红黑边缘配色。

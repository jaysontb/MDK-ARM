# STM32F4 与 LubanCat2 视觉系统联调指南

## 📋 目录
1. [硬件连接](#硬件连接)
2. [软件配置](#软件配置)
3. [测试步骤](#测试步骤)
4. [常见问题](#常见问题)
5. [API使用说明](#api使用说明)

---

## 🔌 硬件连接

### 接线方式

```
┌─────────────┐                    ┌─────────────┐
│  LubanCat2  │                    │  STM32F407  │
│             │                    │             │
│  USART3     │                    │  USART2     │
│   (ttyS3)   │                    │  (115200)   │
│             │                    │             │
│  TX(Pin X) ────────────────────▶ RX(PA3)      │
│  RX(Pin Y) ◀──────────────────── TX(PA2)      │
│  GND       ◀──────┬──────────▶  GND          │
└─────────────┘      │            └─────────────┘
                     │
                     └─ 共地连接(必须!)
```

### 引脚对应表

| 设备 | 引脚 | 功能 | 说明 |
|------|------|------|------|
| **LubanCat2** | USART3 TX | 发送 | 向STM32发送视觉数据 |
| **LubanCat2** | USART3 RX | 接收 | 接收STM32的请求指令 |
| **LubanCat2** | GND | 地 | 与STM32共地 |
| **STM32F407** | PA2 (USART2 TX) | 发送 | 向LubanCat发送请求 |
| **STM32F407** | PA3 (USART2 RX) | 接收 | 接收LubanCat的数据 |
| **STM32F407** | GND | 地 | 与LubanCat共地 |

### ⚠️ 注意事项

1. **交叉连接**: LubanCat TX → STM32 RX, LubanCat RX → STM32 TX
2. **共地连接**: 两个设备的GND必须连接,否则通信不稳定
3. **电平兼容**: 两者均为3.3V逻辑,可直接连接
4. **避免冲突**: 如果使用串口模块调试,调试完成后必须断开电脑连接
5. **供电独立**: LubanCat2和STM32分别供电,不要共用电源

---

## 💻 软件配置

### LubanCat2 端配置

1. **上传视觉程序**
   ```bash
   scp Creama.py lubancat@<lubancat_ip>:/home/lubancat/vision/
   ```

2. **确认串口配置**
   - 打开 `Creama.py` 第 147 行
   - 确认串口为: `ser = serial.Serial(port="/dev/ttyS3", baudrate=115200, timeout=0.05)`

3. **运行视觉程序**
   ```bash
   cd /home/lubancat/vision
   python Creama.py
   ```

4. **预期输出**
   ```
   [INFO] 视觉系统已启动
   [INFO] 串口接收线程运行中,等待MCU指令...
   [INFO] 按 'q' 键退出程序
   ```

### STM32F407 端配置

1. **编译项目**
   - 打开 Keil MDK 或 STM32CubeIDE
   - 编译整个项目(无错误)

2. **烧录程序**
   - 使用 ST-Link 或 J-Link 烧录到STM32

3. **查看调试输出**
   - 连接USART1到电脑(115200, 8N1)
   - 使用串口助手查看测试日志

---

## 🧪 测试步骤

### 阶段1: 基础通信测试

**目标**: 验证STM32能正确发送请求并接收响应

1. **硬件连接**
   - 按上述接线图连接LubanCat2和STM32

2. **启动LubanCat视觉程序**
   ```bash
   python Creama.py
   ```

3. **修改 `main.c` 启用测试**
   ```c
   // 在 main() 函数的 USER CODE BEGIN 2 区域
   Visual_Test_Basic();  // 取消注释这一行
   ```

4. **烧录并运行STM32**
   - 观察USART1输出:
     ```
     === 测试1: 红色物料识别 ===
     红色物料坐标: X=552, Y=230
     
     === 测试2: 红色凸台识别 ===
     红色凸台坐标: X=320, Y=240
     
     === 测试3: 二维码识别 ===
     二维码数据: 123RGB
     ```

5. **验证成功标准**
   - ✅ STM32能成功接收坐标数据
   - ✅ 坐标值合理(在摄像头分辨率范围内)
   - ✅ 无超时错误

### 阶段2: 循环测试(验证稳定性)

**目标**: 长时间连续通信无丢包

1. **修改 `main.c` 启用循环测试**
   ```c
   // 在 while(1) 循环中
   Visual_Test_Loop();  // 取消注释这一行
   ```

2. **运行并观察**
   - 程序会每500ms发送一次请求
   - 观察100次以上通信是否稳定

3. **验证成功标准**
   - ✅ 成功率 > 95%
   - ✅ 无通信卡死现象
   - ✅ 响应时间 < 100ms

### 阶段3: 完整任务流程测试

**目标**: 模拟真实比赛流程

1. **准备测试环境**
   - 放置二维码(如 "123RGB")
   - 放置红色物料块
   - 放置红色凸台

2. **修改 `main.c` 启用完整测试**
   ```c
   Visual_Test_FullTask();  // 取消注释这一行
   ```

3. **预期输出**
   ```
   ======== 开始任务流程 ========
   [步骤1] 读取二维码...
   任务序列: 123RGB
   
   [步骤2] 识别第1个物料...
   物料坐标: X=552, Y=230
   [TODO] 移动到物料位置并抓取...
   
   [步骤3] 定位红色凸台...
   凸台坐标: X=320, Y=240
   [TODO] 移动到凸台位置并放置物料...
   
   ======== 任务完成! ========
   ```

---

## 🐛 常见问题

### 1. STM32 发送请求后无响应

**原因**:
- 硬件连接错误(TX/RX接反)
- LubanCat视觉程序未运行
- 串口配置不匹配

**解决方法**:
```bash
# 在LubanCat上测试串口回环
sudo cat /dev/ttyS3  # 终端1
sudo echo -e "\x66\x02\x01\x77" > /dev/ttyS3  # 终端2

# 应该能在终端1看到发送的数据
```

### 2. 接收到错误的坐标(如 0, 0 或 65535)

**原因**:
- 卡尔曼滤波器初始化问题(已修复)
- 视觉识别失败(未检测到目标)

**解决方法**:
- 确认摄像头画面中有目标物体
- 调整光照条件
- 检查颜色阈值是否准确

### 3. 通信成功率低(丢包严重)

**原因**:
- 没有共地连接
- 串口波特率设置错误
- 线缆过长或干扰严重

**解决方法**:
- 确认GND连接
- 缩短连接线缆(<30cm)
- 远离强电磁干扰源

### 4. LubanCat 收到请求但不响应

**原因**:
- 视觉检测失败(画面中无目标)
- 程序卡死

**解决方法**:
```bash
# 查看LubanCat终端输出
python Creama.py  # 观察是否有 "[UART] 收到指令" 日志

# 如果有日志但无响应,检查画面
DEBUG_VISUAL = True  # 开启调试窗口
```

### 5. 编译错误: undefined reference to Visual_XXX

**原因**: Keil工程未包含新添加的源文件

**解决方法**:
1. 右键项目 → Options for Target
2. C/C++ → Include Paths: 添加 `../Inc`
3. 右键源文件组 → Add Existing Files: 添加 `visual_test.c`

---

## 📚 API使用说明

### 发送函数

#### 1. 请求识别物料
```c
/**
 * @brief  请求识别指定颜色的物料块
 * @param  color: COLOR_RED(1), COLOR_GREEN(2), COLOR_BLUE(3)
 */
Visual_Send_Material_Request(COLOR_RED);
```

#### 2. 请求识别凸台
```c
/**
 * @brief  请求识别指定颜色的凸台
 * @param  color: COLOR_RED(1), COLOR_GREEN(2), COLOR_BLUE(3)
 */
Visual_Send_Platform_Request(COLOR_GREEN);
```

#### 3. 请求识别凹槽
```c
/**
 * @brief  请求识别指定颜色的凹槽
 * @param  color: COLOR_RED(1), COLOR_GREEN(2), COLOR_BLUE(3)
 */
Visual_Send_Slot_Request(COLOR_BLUE);
```

#### 4. 请求识别二维码
```c
/**
 * @brief  请求识别二维码
 */
Visual_Send_QRCode_Request();
```

### 接收处理

#### 完整示例代码
```c
// 在main函数中

// 初始化
Visual_Data_Init();
Visual_UART_Start_Receive();

while(1)
{
    // 1. 发送请求
    Visual_Send_Material_Request(COLOR_RED);
    
    // 2. 等待响应(最多500ms)
    uint32_t start = HAL_GetTick();
    while ((HAL_GetTick() - start) < 500)
    {
        if (visual_rx_complete_flag)  // 接收完成
        {
            // 3. 解析数据
            Visual_Data_Unpack(Visual_Get_RxBuffer());
            Visual_Clear_RxFlag();
            
            // 4. 使用数据
            uint16_t x = VIS_RX.r_block_x;
            uint16_t y = VIS_RX.r_block_y;
            
            // TODO: 控制电机移动到 (x, y)
            
            break;
        }
    }
    
    HAL_Delay(1000);
}
```

### 数据结构

```c
// 全局变量 VIS_RX 包含所有视觉数据
extern VisualRxData_t VIS_RX;

// 访问示例:
VIS_RX.r_block_x;      // 红色物料X坐标
VIS_RX.r_block_y;      // 红色物料Y坐标
VIS_RX.g_block_x;      // 绿色物料X坐标
VIS_RX.b_block_x;      // 蓝色物料X坐标
VIS_RX.platform_x;     // 凸台X坐标
VIS_RX.platform_y;     // 凸台Y坐标
VIS_RX.slot_x;         // 凹槽X坐标
VIS_RX.slot_y;         // 凹槽Y坐标
VIS_RX.qr_data[6];     // 二维码数据(6个ASCII字符)
VIS_RX.qr_valid;       // 二维码数据有效标志(1=有效)
```

---

## 🎯 下一步工作

1. **集成到业务逻辑**
   - 将视觉坐标转换为机械坐标系
   - 编写运动控制函数
   - 实现抓取和放置逻辑

2. **参数调优**
   - 根据实际测试调整颜色阈值
   - 校准坐标系映射关系
   - 优化识别准确率

3. **异常处理**
   - 添加超时重发机制
   - 处理识别失败情况
   - 实现错误恢复策略

---

## 📞 支持

如有问题,请检查:
1. 硬件连接是否正确
2. 两端程序是否都在运行
3. 串口配置是否一致
4. 目标物体是否在摄像头视野内

祝联调顺利! 🚀

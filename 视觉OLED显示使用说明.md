# 视觉系统OLED显示使用说明

## 📋 修改内容

已将所有视觉测试函数从USART1输出改为OLED显示,因为USART1用于控制电机。

---

## 🎯 可用函数

### 1. **Visual_Show_Coord_OLED() - 显示坐标**

在OLED指定行显示坐标。

```c
void Visual_Show_Coord_OLED(uint16_t x, uint16_t y, uint8_t line);

// 使用示例:
Visual_Show_Coord_OLED(515, 204, 2);  // 在第2行显示 X:515 Y:204
```

**OLED显示效果:**
```
X:515  Y:204
```

---

### 2. **Visual_Test_Basic() - 基础测试**

测试物料和凸台识别,结果显示在OLED上。

```c
// 在main.c中调用:
Visual_Test_Basic();
```

**OLED显示流程:**
```
第1步:
┌────────────────┐
│Vision Test     │  ← 标题
│Test1:Material  │  ← 测试项目
│X:515  Y:204    │  ← 物料坐标
│                │
└────────────────┘

第2步(自动切换):
┌────────────────┐
│Test2:Platform  │  ← 测试项目
│X:515  Y:205    │  ← 凸台坐标
│Success!        │  ← 状态
│                │
└────────────────┘
```

---

### 3. **Visual_Test_Loop() - 循环测试**

持续循环测试三种颜色的物料,OLED实时更新。

```c
// 在main.c的while(1)中调用:
while (1)
{
    Visual_Test_Loop();
    HAL_Delay(500);
}
```

**OLED显示效果:**
```
┌────────────────┐
│Loop Test R:001 │  ← 测试次数
│X:515  Y:204    │  ← 当前坐标
│OK              │  ← 状态
│                │
└────────────────┘
```

R/G/B 自动轮换显示。

---

### 4. **Visual_Display_All_Info() - 显示所有信息**

在OLED上显示所有视觉数据(物料、凸台、凹槽)。

```c
// 实时监控视觉状态:
Visual_Display_All_Info();
```

**OLED显示效果:**
```
┌────────────────┐
│Vision Monitor  │  ← 标题
│R:X:515  Y:204  │  ← 红色物料坐标
│P:X:515  Y:205  │  ← 凸台坐标
│S:X:---  ---    │  ← 凹槽(未检测)
└────────────────┘
```

**说明:**
- `R:` 红色物料 (r_block)
- `P:` 凸台 (platform)
- `S:` 凹槽 (slot)
- `---` 表示未检测到

---

### 5. **Visual_Quick_Test() - 快速测试 ⭐推荐**

单次快速测试,返回成功/失败状态,**适合集成到运动控制流程**。

```c
uint8_t Visual_Quick_Test(uint8_t color, uint8_t line);

// 使用示例:
if (Visual_Quick_Test(COLOR_RED, 2))
{
    // 成功,可以使用 VIS_RX.r_block_x, VIS_RX.r_block_y
    // 调用运动控制函数移动到该坐标
    Motion_MoveTo(VIS_RX.r_block_x, VIS_RX.r_block_y);
}
else
{
    // 识别失败,进行错误处理
    OLED_ShowString(3, 1, "Retry...");
}
```

**OLED显示:**
```
等待时:  R:Wait..
成功时:  R:X:515  Y:204
失败时:  R:Error
```

---

## 💡 集成到运动控制的完整示例

```c
void Task_GrabRedBlock(void)
{
    OLED_Clear();
    OLED_ShowString(1, 1, "Grab Red Block");
    
    // 步骤1: 识别红色物料
    if (!Visual_Quick_Test(COLOR_RED, 2))
    {
        OLED_ShowString(3, 1, "Vision Failed!");
        return;
    }
    
    // 步骤2: 移动到物料位置
    OLED_ShowString(3, 1, "Moving...");
    Motion_MoveTo_Pixel(VIS_RX.r_block_x, VIS_RX.r_block_y);
    
    // 步骤3: 抓取物料
    OLED_ShowString(3, 1, "Grabbing...");
    Gripper_Close();
    HAL_Delay(500);
    Gripper_Lift();
    
    // 步骤4: 识别红色凸台
    Visual_Send_Platform_Request(COLOR_RED);
    if (Visual_Wait_Response(500))
    {
        Visual_Data_Unpack(Visual_Get_RxBuffer());
        
        // 步骤5: 移动到凸台并放置
        OLED_ShowString(4, 1, "Placing...");
        Motion_MoveTo_Pixel(VIS_RX.platform_x, VIS_RX.platform_y);
        Gripper_Open();
        
        OLED_ShowString(4, 1, "Done!");
    }
}
```

---

## 🔧 main.c中的调用示例

```c
#include "visual_test.h"
#include "oled.h"

int main(void)
{
    /* 硬件初始化 */
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_I2C1_Init();
    MX_USART2_UART_Init();  // 视觉通信
    
    /* OLED初始化 */
    OLED_Init();
    OLED_Clear();
    OLED_ShowString(1, 1, "System Ready");
    HAL_Delay(1000);
    
    /* 视觉通信初始化 */
    Visual_UART_Init();
    
    /* 方式1: 单次测试 */
    Visual_Test_Basic();
    HAL_Delay(3000);
    
    /* 方式2: 实时监控 */
    while (1)
    {
        Visual_Display_All_Info();
        HAL_Delay(1000);
    }
    
    /* 方式3: 快速测试(推荐用于任务流程) */
    while (1)
    {
        if (Visual_Quick_Test(COLOR_RED, 2))
        {
            // 识别成功,执行后续动作
            // Motion_MoveTo(...);
        }
        HAL_Delay(500);
    }
}
```

---

## 📊 OLED行号说明

OLED共4行,每行16个字符:

```
行1: ┌────────────────┐  ← 标题/状态
行2: │                │  ← 数据1
行3: │                │  ← 数据2
行4: └────────────────┘  ← 数据3/状态
```

---

## ⚠️ 注意事项

1. **USART1已用于电机控制**,所有调试输出都通过OLED
2. **视觉通信使用USART2**,波特率115200
3. **坐标数据存储在全局变量** `VIS_RX` 中:
   ```c
   VIS_RX.r_block_x, VIS_RX.r_block_y  // 红色物料
   VIS_RX.g_block_x, VIS_RX.g_block_y  // 绿色物料
   VIS_RX.b_block_x, VIS_RX.b_block_y  // 蓝色物料
   VIS_RX.platform_x, VIS_RX.platform_y  // 凸台
   VIS_RX.slot_x, VIS_RX.slot_y  // 凹槽
   ```

4. **OLED显示会覆盖之前的内容**,根据需要调用 `OLED_Clear()`

---

## ✅ 编译测试

修改完成后:
1. 编译项目(无错误)
2. 下载到STM32
3. 确保LubanCat2视觉服务运行中
4. 观察OLED显示

如果OLED显示 `ERROR:Timeout`,检查:
- LubanCat2是否开机并运行视觉服务
- USART2连接是否正常(TX2→RX_LubanCat, RX2→TX_LubanCat)
- 波特率是否为115200

---

**修改完成!现在可以专注于运动控制开发了!** 🚀

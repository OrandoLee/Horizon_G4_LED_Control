# Horizon_G4_UART_Control

## 1. 项目简介

**Horizon_G4_UART_Control** 是 Horizon-G4 计划的第二个里程碑。

本项目在 STM32G431 高性能开发板上，实现了基于 **USART1** 的双向串行通信系统。通过上位机（VOFA+）发送字符指令，实时控制板载 LED 的亮灭，并利用重定向的 `printf` 函数回传设备状态，构成了完整的**“指令-执行-反馈”**闭环链路。

## 2. 硬件架构与连接

### 2.1 核心硬件

- **MCU**: STM32G431RBT6 (Arm Cortex-M4)
- **主频**: 170MHz (超频性能配置)
- **外设**: 板载 LED (PC8~PC15) + 锁存器 (74HC573)

### 2.2 物理连接 (USART1)

使用 USB-TTL 串口模块与开发板通信：

- **PA9 (TX) ** $\leftrightarrow$ 模块 RX
- **PA10 (RX)** $\leftrightarrow$ 模块 TX
- **GND** $\leftrightarrow$ 模块 GND (**必须共地**)

### 2.3 LED 控制逻辑

- **数据总线**: PC8 (Low = 亮, High = 灭)
- **控制闸门**: PD2 (High = 开闸, Low = 关闸/锁存)

## 3. 软件配置参数 (CubeMX)

### 3.1 时钟树 (Clock Tree)

- **HSE**: 24MHz (外部晶振)
- **PLLM**: /6 (4MHz 输入 PLL)
- **PLLN**: x85 (340MHz VCO)
- **PLLR**: /2
- **HCLK**: **170MHz**

### 3.2 串口配置 (Connectivity)

- **Instance**: **USART1**
- **Mode**: Asynchronous (异步)
- **Baud Rate**: **115200** Bits/s
- **Word Length**: 8 Bits
- **Parity**: None
- **Stop Bits**: 1

### 3.3 GPIO 设置

- **PD2**: GPIO_Output (Initial: Low)
- **PC8~PC15**: GPIO_Output (Initial: High) -> *上电默认灭灯*

## 4. 核心代码逻辑

### 4.1 关键特性

1. **初始化复位**：在 `while(1)` 前强制执行灭灯操作，防止上电 LED 乱闪。
2. **重定向 printf**：实现了 `fputc` 函数，支持通过 `printf` 直接向串口打印调试信息。
3. **阻塞式接收**：使用 `HAL_UART_Receive` 配合 10ms 超时，兼顾实时性与 CPU 占用。

### 4.2 代码片段

C

```
/* 重定向 printf 到 USART1 */
int fputc(int ch, FILE *f) {
  HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, 0xffff);
  return ch;
}

/* 主循环逻辑 */
while (1) {
  if(HAL_UART_Receive(&huart1, &rx_data, 1, 10) == HAL_OK) {
    HAL_GPIO_WritePin(GPIOD, GPIO_PIN_2, GPIO_PIN_SET); // 开闸
    
    if(rx_data == '1') {
       HAL_GPIO_WritePin(GPIOC, GPIO_PIN_8, GPIO_PIN_RESET); // 点亮
       printf("LED ON\r\n"); // 反馈
    }
    else if(rx_data == '0') {
       HAL_GPIO_WritePin(GPIOC, GPIO_PIN_8, GPIO_PIN_SET);   // 熄灭
       printf("LED OFF\r\n"); // 反馈
    }
    
    HAL_GPIO_WritePin(GPIOD, GPIO_PIN_2, GPIO_PIN_RESET); // 关闸
  }
}
```

## 5. 调试指南 (VOFA+)

### 5.1 上位机设置

- **软件**: VOFA+ (v1.3.10+)
- **协议**: RawData
- **接口**: 这里的 COM 口号, 波特率 **115200**

### 5.2 发送指令

- **开灯**: 发送字符 `1` (发送模式选 String)
  - *预期反馈*: `LED ON`
- **关灯**: 发送字符 `0` (发送模式选 String)
  - *预期反馈*: `LED OFF`

### 5.3 常见坑点排查

- **灯不亮**: 检查 Keil 中是否勾选了 **Use MicroLIB** (printf 必需)。
- **乱码/无反应**: 检查时钟输入是否误设为 8MHz (实际应为 24MHz)。
- **一直亮**: 检查是否在 `while(1)` 之前添加了灭灯初始化代码。


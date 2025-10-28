[firewater | VOFA-Plus上位机](https://www.vofa.plus/docs/learning/dataengines/firewater)
## 一、重点说明

FireWater 协议**必须遇到换行才会打印数据**。

## 二、协议特点

1. **格式类型**：CSV 风格的字符串流，直观简洁，编程难度类似`printf`。
2. **局限性**：字符串解析会消耗较多运算资源（上位机和下位机均如此），因此**仅建议在通道数量不多、发送频率不高时使用**。

## 三、采样数据解析

### 1. 数据格式

- 基本格式：`<any>:ch0,ch1,ch2,...,chN\n`
- 关键规则：
    - `any`和冒号可以省略，但**换行符（\n）不可省略**。
    - `any`不能为 "image"（该前缀用于解析图片数据）。
    - 换行符可以是`\n`、`\n\r`或`\r\n`（实际换行，非字符 “\”+“n”）。
- 示例：
    - 带标识：`"channels: 1.386578,0.977929,-0.628913,-0.942729\n"`
    - 不带标识：`"1.386578,0.977929,-0.628913,-0.942729\n"`

### 2. Arduino 示例代码

```cpp
void setup() {
  Serial.begin(115200);
}
float t = 0;
void loop() {
  t += 0.1;
  // 带标识的发送方式
  Serial.print("samples:%f, %f, %f, %f\n", sin(t), sin(2*t), sin(3*t), sin(4*t));
  // 不带标识的发送方式（也可使用）
  // Serial.print("%f, %f, %f, %f\n", sin(t), sin(2*t), sin(3*t), sin(4*t));
  delay(100);
}
```

## 四、图片解析

### 1. 数据格式

- 需先发送**前导帧**，再发送图片数据：
    - 前导帧格式：`"image:IMG_ID, IMG_SIZE, IMG_WIDTH, IMG_HEIGHT, IMG_FORMAT\n"`
        - `image:`：协议约定的图片前导帧开头，用于标识图片数据。
        - `IMG_ID`：图片通道 ID，用于区分不同图片。
        - `IMG_SIZE`：即将发送的图片尺寸。
        - `IMG_WIDTH`：图片宽度。
        - `IMG_HEIGHT`：图片高度。
        - `IMG_FORMAT`：图片格式（如 8 位灰度图、16 位灰度图、jpg 等）。
    - 后续发送真正的图片数据，协议会根据前导帧的`IMG_FORMAT`解析。

### 2. Arduino 示例代码
```cpp
void setup() {
  Serial.begin(115200);
}
void loop() {
  // 发送前导帧
  Serial.write("image:%d,%d,%d,%d,%d\n",
    IMG_ID,      // 图片通道ID
    IMG_SIZE,    // 图片数据大小
    IMG_WIDTH,   // 图片宽度
    IMG_HEIGHT,  // 图片高度
    IMG_FORMAT   // 图片格式
  );
  // 发送图片数据
  Serial.write(IMG_DATA, IMG_LENGTH);
}
```
## 五、文本打印相关说明

1. **解析触发**：以换行作为帧结束标志，遇到换行才开启一帧解析（判断为采样数据帧、图片前导帧或其他数据）；未解析时不打印文本。
2. **数据包定义**：图片前导帧加后续图片数据称为 “图片数据包”，图片数据会缩略打印。
3. **显示设置**：点击字节接收区设置按钮，可单独隐藏采样数据帧、图片数据包，或隐藏包括其他数据在内的所有数据。
4. **注意事项**：若发送数据一直无换行，会导致缓冲区爆满、软件卡死。


# UART 监控模块（FireWater 协议实现）笔记

## 一、模块核心功能

该 Verilog 模块`uart_monitor_top`实现了基于**FireWater 协议**的 UART 数据发送功能，将 7 个 16 位有符号数据转换为 CSV 风格的字符串流，通过 UART 串口发送，满足 FireWater 协议 “字符串流 + 换行结束” 的核心要求。

## 二、与 FireWater 协议的映射关系

FireWater 协议的核心是 “CSV 风格字符串流，以换行（\n）作为帧结束标志”，模块通过以下设计实现协议要求：

1. **帧格式匹配**
    
    - 发送数据格式：`数据0, 数据1, 数据2, 数据3, 数据4, 数据5, 数据6\n`
        - 各数据间用逗号（`8'h2C`）分隔，最后一个数据后接换行符（`8'h0A`），符合 FireWater “换行触发解析” 的规则。
        - 每个数据经`itoa`模块转换为 ASCII 字符串（数字字符），整体构成字符串流，符合协议 “CSV 风格” 特性。
2. **字符串流发送**
    
    - 模块将 7 个 16 位有符号整数依次转换为 ASCII 字符，通过 UART 串行发送，最终形成完整的 FireWater 帧（以换行结束）。

## 三、模块关键参数与接口

### 1. 参数

- `sys_clk_DIV`：UART 波特率分频系数（如 50MHz 时钟下，`434`对应 115200 波特率）。

### 2. 接口

|信号名|方向|说明|
|---|---|---|
|`sys_rst_n`|输入|系统复位（低电平有效）|
|`sys_clk`|输入|系统时钟|
|`i_en`|输入|数据发送使能（高电平触发发送）|
|`i_val0`-`i_val6`|输入|7 个 16 位有符号待发送数据|
|`i_uart_rx`|输入|UART 接收引脚|
|`o_uart_tx`|输出|UART 发送引脚|

## 四、核心逻辑设计

### 1. 主控制状态机

状态机控制数据发送全流程，确保按 FireWater 格式依次处理 7 个数据：

|状态|功能描述|
|---|---|
|`IDLE`|空闲状态，等待`i_en`使能信号触发发送流程|
|`SELECT`|依次选择 7 个数据通道，为每个数据启动`itoa`转换，并设置分隔符（逗号 / 换行）|
|`WAIT`|等待`itoa`转换模块准备就绪|
|`PARSING`|等待`itoa`转换完成（将整数转为 ASCII 字符串）|
|`SENDING`|通过 UART 发送当前数据的 ASCII 字符串（含分隔符），完成后切换到下一个数据|

### 2. 关键子模块 / 信号

#### （1）`itoa`转换模块

- 功能：将 16 位有符号整数转换为 ASCII 字符串（6 字节数组`itoa_str`），支持负数（含符号位`-`）。
- 转换逻辑：
    - 提取符号位（`itoa_sign`），计算绝对值（`itoa_abs`）；
    - 逐位取模（除以 10），将每一位数字转为 ASCII 字符（`0x30`-`0x39`对应`0`-`9`）；
    - 结果存于`itoa_str`，最终拼接为完整数字字符串。

#### （2）UART 发送模块

- 功能：将`itoa`转换后的 ASCII 字符串（含分隔符）按波特率串行发送。
- 发送逻辑：
    - 以 “起始位 + 8 数据位 + 停止位” 格式发送；
    - 用`tx_cnt`计数发送位数，`ccnt`分频时钟生成波特率；
    - `tx_rdy`信号指示发送就绪，确保字符连续发送。

#### （3）通道使能控制

- 用`channel_enable`（7 位寄存器）控制各通道是否发送：
    - 接收 UART 命令（如`'A'`使能全部通道，`'C'`关闭全部，`'0'`-`'6'`单独切换通道）；
    - 关闭的通道发送`0`（通过`itoa_val`赋值控制）。

## 五、FireWater 协议关键细节实现

1. **换行触发解析**
    
    - 第 7 个数据（最后一个）的分隔符设置为换行符（`8'h0A`），确保整个帧以换行结束，符合 FireWater “遇到换行才解析” 的规则。
2. **字符串流格式**
    
    - 每个数据的 ASCII 字符串 + 空格 + 分隔符（逗号 / 换行）构成连续字符流，无额外帧头 / 帧尾，符合协议 “简洁 CSV 风格”。
3. **局限性适配**
    
    - FireWater 协议不适合高频率 / 多通道场景，本模块仅支持 7 个通道，且通过状态机依次发送，适配协议特性。

## 六、注意事项

- 若`i_en`持续有效，模块会循环发送 7 个数据（需外部控制`i_en`的触发频率，避免数据溢出）；
- `itoa`转换模块支持 6 字节字符串（含符号位），需确保输入数据范围在转换能力内（避免溢出）；
- UART 接收命令仅控制通道使能，不影响 FireWater 帧格式本身。


# FireWater 协议实现常见错误与避坑笔记

## 一、FireWater 协议核心要求（必须牢记）

1. **数据格式**：严格遵循 “CSV 风格的十进制 ASCII 字符串流”，格式为：  
    `数值1, 数值2, ..., 数值N\n`
    
    - 每个 “数值” 必须是**十进制 ASCII 字符串**（如`123`→`"123"`，`-456`→`"-456"`）。
    - 通道间用逗号（`,`，ASCII `0x2C`）分隔。
    - 整帧必须以换行符（`\n`，ASCII `0x0A`）结束（上位机仅在收到换行时解析）。
2. **核心原则**：协议不关心数据 “如何生成”，只关心 “是否能被解析为一串带分隔符的十进制数值”。
    

## 二、常见错误类型及避坑要点

### 1. 数值转换错误（最核心错误）

#### 错误表现：

- 直接拆分二进制位生成字符（如取`triangle_abs[11:8]`转换为 ASCII），本质是 “二进制位的字符化”，而非 “数值的十进制字符串化”。  
    例：数值`123`（二进制`1111011`）被拆分为高 4 位`7`（`0x37`→`'7'`）和低 4 位`11`（`0x3B`→非数字字符），生成无意义的`"7;"`，而非正确的`"123"`。

#### 避坑要点：

- **必须实现 “数值→十进制 ASCII” 转换**：
    - 无符号数（如`0~4095`）：逐位取模（除以 10），将每一位数字转为 ASCII（`0x30`→`'0'`，`0x39`→`'9'`）。
    - 有符号数（如`-2048~2047`）：先处理符号位（负数加`'-'`），再转换绝对值。
- 推荐使用封装好的转换函数（如`unsigned_to_ascii`、`signed_to_ascii`），避免手动拆分位。

### 2. 帧格式混乱（直接导致解析失败）

#### 错误表现：

- 分隔符与通道不匹配（部分通道用逗号，部分用其他符号）。
- 缺少换行符或换行符位置错误（如在中间通道后换行）。
- 通道数据长度不一致（有的通道 1 个字符，有的通道 5 个字符，无规律）。

#### 避坑要点：

- **严格遵循 “逗号分隔 + 换行结束”**：
    - 前`N-1`个通道后必须加逗号（`,`），第`N`个通道后必须加换行符（`\n`）。
- **动态计算帧长度**：
    - 帧长度 = 所有通道的 ASCII 字符串长度之和 + 逗号数量（`N-1`） + 1（换行符）。
    - 例：8 通道数据，每个通道字符串平均 3 个字符，则总长度≈`8×3 +7 +1=32`字节。

### 3. 通道逻辑与参数不匹配

#### 错误表现：

- 参数`CHANNEL_COUNT`设为 8，但实际处理的通道数不足 8，或部分通道数据无意义（如固定发送`'3'`）。
- 通道数据与物理意义脱节（如用随机位填充，而非实际传感器 / 信号数据）。

#### 避坑要点：

- **通道数与参数严格对应**：
    - 若`CHANNEL_COUNT=8`，必须显式处理 8 个通道，每个通道的数据需关联实际物理量（如 DDS 输出、传感器读数）。
- **数据需有实际意义**：
    - 确保每个通道的数值是 “可被监控的有效数据”（如波形幅值、计数器值），避免无意义的随机数。

### 4. 冗余逻辑干扰

#### 错误表现：

- 保留未使用的代码（如`JustFloat`协议相关信号、未启用的接收逻辑），导致逻辑混乱，甚至意外干扰发送流程。

#### 避坑要点：

- **精简代码，只保留必要逻辑**：
    - 若不使用接收功能，直接删除接收相关信号（如`rx_buffer`、`rx_state`），避免冗余变量占用资源或引发误操作。

## 三、验证与调试技巧（自查工具）

1. **输出监控**：用串口助手查看发送的原始数据（十六进制 / ASCII 模式），检查：
    - 是否有换行符`0x0A`结尾。
    - 逗号`0x2C`是否仅出现在通道之间。
    - 字符是否均为十进制相关（`0-9`、`-`）。
2. **单步测试**：
    - 先测试 1 个通道，确保能正确发送`"123\n"`，上位机能解析。
    - 再逐步增加到`N`个通道，验证分隔符和换行符是否正确。
3. **转换函数单独验证**：
    - 给转换函数输入已知值（如`123`、`-45`），检查输出是否为`"123"`、`"-45"`。

## 四、核心原则总结

1. **“转换” 是灵魂**：没有正确的 “数值→十进制 ASCII” 转换，就不可能符合 FireWater 协议。
2. **“格式” 是底线**：哪怕数据正确，格式错误（如缺换行符）也会导致上位机无法解析。
3. **“简洁” 是保障**：冗余逻辑越少，出错概率越低，优先保证核心功能（转换 + 发送）正确。


代码如下：
```verilog
//UART数据监控顶层模块，将7个16位有符号数据通过UART串口发送  
module uart_monitor_top#(  
    parameter [15:0] sys_clk_DIV = 434 // UART分频倍率，例如若时钟频率为50MHZ, sys_clk_DIV=434, 则UART波特率为115200  
)  
(  
    // 系统控制信号  
    input wire          sys_rst_n,      // 系统复位信号，低电平有效  
    input wire          sys_clk,        // 系统时钟信号  
    input wire          i_en,           // 数据发送使能信号  
      
    // 数据输入信号 - 7个16位有符号数据  
    input wire signed [15:0] i_val0,     
    input wire signed [15:0] i_val1,     
    input wire signed [15:0] i_val2,     
    input wire signed [15:0] i_val3,     
    input wire signed [15:0] i_val4,     
    input wire signed [15:0] i_val5,     
    input wire signed [15:0] i_val6,     
  
     // UART输入输出信号  
    input wire          i_uart_rx,        
    output reg          o_uart_tx         
);  
  
// UART TX信号初始化为高电平（空闲状态）  
initial o_uart_tx = 1'b1;  
  
  
// 状态机定义 - 主控制状态机  
localparam [2:0] IDLE    = 3'd0,       // 空闲状态，等待使能信号  
                 SELECT  = 3'd1,       // 选择状态，选择要发送的数据  
                 WAIT    = 3'd2,       // 等待状态，等待itoa转换准备  
                 PARSING = 3'd3,       // 解析状态，等待itoa转换完成  
                 SENDING = 3'd4;       // 发送状态，通过UART发送数据  
  
// 信号声明  
//==============================================================================  
// 状态机控制信号  
reg  [2:0] stat;                       // 当前状态寄存器  
reg  [6:0] channel_enable;             // 7个通道的独立使能控制（bit0对应通道0，bit6对应通道6）  
// UART接收控制信号  
wire            rx_valid;              // UART接收数据有效信号  
wire [7:0]      rx_data;               // UART接收数据  
reg  [7:0]      rx_cmd_buffer;         // 接收命令缓存  
reg  [2:0]      rx_channel_select;     // 接收到的通道选择  
reg             rx_cmd_valid;          // 命令有效标志  
// UART发送控制信号  
wire tx_rdy;                           // UART发送就绪信号  
reg  tx_en;                            // UART发送使能信号  
reg  [7:0] tx_data;                    // UART发送数据  
  
// itoa转换模块信号  
reg          itoa_en;                  // itoa转换使能信号  
reg signed [15:0] itoa_val;            // itoa转换输入值  
reg          itoa_oen;                 // itoa转换输出使能信号  
reg  [7:0] itoa_str[0:5];              // itoa转换输出字符串数组（6字节）  
  
// 数据处理控制信号  
reg  [3:0] vcnt;                       // 数据选择计数器（0-6，对应7个输入数据）  
reg  [2:0] cnt;                        // 字符发送计数器（0-7，对应8个字符）  
reg  [7:0] eov;                        // 结束字符（逗号或换行）  
  
// 字符串组合信号  
wire [7:0] s_str[0:7];                 // 完整的8字节发送字符串  
initial begin  
    channel_enable = 7'b1111111;        // 默认所有通道使能  
end  
  
// 字符串组合逻辑，将itoa转换结果与分隔符组合  
assign s_str[0] = itoa_str[0];         // 数字字符0（最低位）  
assign s_str[1] = itoa_str[1];         // 数字字符1  
assign s_str[2] = itoa_str[2];         // 数字字符2  
assign s_str[3] = itoa_str[3];         // 数字字符3  
assign s_str[4] = itoa_str[4];         // 数字字符4  
assign s_str[5] = itoa_str[5];         // 数字字符5（符号位或最高位）  
assign s_str[6] = 8'h20;               // 空格字符  
assign s_str[7] = eov;                 // 结束字符（逗号或换行）  
  
// UART接收  
always @(posedge sys_clk or negedge sys_rst_n) begin  
    if (~sys_rst_n) begin  
        channel_enable <= 7'b1111111;   // 复位时默认所有通道使能  
        rx_cmd_buffer <= 8'h00;  
        rx_channel_select <= 3'd0;  
        rx_cmd_valid <= 1'b0;  
    end else begin  
        rx_cmd_valid <= 1'b0;            // 默认命令无效  
          
        if (rx_valid) begin               // 接收到有效数据  
            rx_cmd_buffer <= rx_data;  
              
            // 解析接收到的命令  
            case (rx_data)  
                // 全局控制命令  
                8'h41: channel_enable <= 7'b1111111;  // 'A' - 全部使能  
                8'h43: channel_enable <= 7'b0000000;  // 'C' - 全部关闭  
                  
                // 单字节通道控制命令  
                8'h30: channel_enable[0] <= ~channel_enable[0];  // '0' - 切换通道0  
                8'h31: channel_enable[1] <= ~channel_enable[1];  // '1' - 切换通道1  
                8'h32: channel_enable[2] <= ~channel_enable[2];  // '2' - 切换通道2  
                8'h33: channel_enable[3] <= ~channel_enable[3];  // '3' - 切换通道3  
                8'h34: channel_enable[4] <= ~channel_enable[4];  // '4' - 切换通道4  
                8'h35: channel_enable[5] <= ~channel_enable[5];  // '5' - 切换通道5  
                8'h36: channel_enable[6] <= ~channel_enable[6];  // '6' - 切换通道6  
                  
                default: begin  
                    // 无效命令，不做任何操作  
                end  
            endcase  
        end  
    end  
end  
  
// UART发送控制逻辑 - 组合逻辑  
always @ (*) begin  
    tx_en = 1'b0;                      // 默认不发送  
    tx_data = 0;                       // 默认发送数据为0  
      
    // 在发送状态时直接发送数据（不再检查通道使能）  
    if(stat == SENDING) begin  
        tx_en = 1'b1;                  // 使能UART发送  
        tx_data = s_str[cnt];          // 发送当前字符  
    end  
end  
  
  
// 主状态机 - 控制整个数据发送流程  
always @ (posedge sys_clk or negedge sys_rst_n) begin  
    if(~sys_rst_n) begin  
        // 复位时初始化所有信号  
        stat <= IDLE;  
        itoa_en <= 1'b0;  
        itoa_val <= 0;  
        vcnt <= 0;  
        cnt <= 0;  
        eov <= 8'h20;                  // 默认结束字符为空格  
    end else begin  
        itoa_en <= 1'b0;               // 默认关闭itoa使能  
        case(stat)  
            // 空闲状态：等待使能信号  
            IDLE: begin  
                if(i_en)  
                    stat <= SELECT;  
            end  
            // 选择状态：依次选择7个输入数据进行处理  
            SELECT: begin  
                if (vcnt == 4'd0) begin  
                    vcnt <= vcnt + 4'd1;  
                    stat <= WAIT;  
                    itoa_en <= 1'b1;  
                    itoa_val <= channel_enable[0] ? i_val0 : 16'd0;  // 通道关闭时输出0  
                    eov <= 8'h2C;          // 逗号分隔符  
                end else if(vcnt == 4'd1) begin  
                    vcnt <= vcnt + 4'd1;  
                    stat <= WAIT;  
                    itoa_en <= 1'b1;  
                    itoa_val <= channel_enable[1] ? i_val1 : 16'd0;  // 通道关闭时输出0  
                    eov <= 8'h2C;  
                end else if(vcnt == 4'd2) begin  
                    vcnt <= vcnt + 4'd1;  
                    stat <= WAIT;  
                    itoa_en <= 1'b1;  
                    itoa_val <= channel_enable[2] ? i_val2 : 16'd0;  // 通道关闭时输出0  
                    eov <= 8'h2C;  
                end else if(vcnt == 4'd3) begin  
                    vcnt <= vcnt + 4'd1;  
                    stat <= WAIT;  
                    itoa_en <= 1'b1;  
                    itoa_val <= channel_enable[3] ? i_val3 : 16'd0;  // 通道关闭时输出0  
                    eov <= 8'h2C;  
                end else if(vcnt == 4'd4) begin  
                    vcnt <= vcnt + 4'd1;  
                    stat <= WAIT;  
                    itoa_en <= 1'b1;  
                    itoa_val <= channel_enable[4] ? i_val4 : 16'd0;  // 通道关闭时输出0  
                    eov <= 8'h2C;  
                end else if(vcnt == 4'd5) begin  
                    vcnt <= vcnt + 4'd1;  
                    stat <= WAIT;  
                    itoa_en <= 1'b1;  
                    itoa_val <= channel_enable[5] ? i_val5 : 16'd0;  // 通道关闭时输出0  
                    eov <= 8'h2C;  
                end else if(vcnt == 4'd6) begin  
                    vcnt <= vcnt + 4'd1;  
                    stat <= WAIT;  
                    itoa_en <= 1'b1;  
                    itoa_val <= channel_enable[6] ? i_val6 : 16'd0;  // 通道关闭时输出0  
                    eov <= 8'h0A;          // 换行符（最后一个数据后）  
                end else begin  
                    // 所有数据处理完成，返回空闲状态  
                    vcnt <= 3'd0;  
                    stat <= IDLE;  
                    eov <= 8'h20;  
                end  
            end  
            // 等待状态：等待一个时钟周期让itoa模块准备  
            WAIT: begin  
                stat <= PARSING;  
            end  
            // 解析状态：等待itoa转换完成  
            PARSING: begin  
                if(itoa_oen)                // itoa转换完成， 若注释则直接跳转到SENDING状态，不等待itoa转换完成  
                    stat <= SENDING;  
            end  
            // 发送状态：通过UART发送8个字符  
            default: begin // SENDING状态  
                if(tx_rdy) begin            // UART发送就绪  
                    cnt <= cnt + 3'd1;     // 移动到下一个字符  
                    if(cnt == 3'd7)        // 8个字符发送完成  
                        stat <= SELECT;    // 返回选择状态处理下一个数据  
                end  
            end  
        endcase  
    end  
end  
  
  
// itoa转换模块 - 将16位有符号整数转换为ASCII字符串  
reg [2:0] itoa_cnt;                    // itoa转换步骤计数器  
reg       itoa_sign;                   // 符号位标志  
reg       itoa_zero;                   // 零值标志  
reg [15:0] itoa_abs;                   // 绝对值寄存器  
wire[15:0] itoa_rem_w = (itoa_abs % 16'd10); // 取模运算（获取个位数）  
reg [3:0] itoa_rem;                    // 余数寄存器  
  
// itoa转换主逻辑  
always @ (posedge sys_clk or negedge sys_rst_n) begin  
    if(~sys_rst_n) begin  
        // 复位时初始化所有itoa相关信号  
        itoa_cnt <= 3'd0;  
        {itoa_sign, itoa_abs, itoa_zero, itoa_rem} <= 0;  
        itoa_oen <= 1'b0;  
        itoa_str[0] <= 0;  
        itoa_str[1] <= 0;  
        itoa_str[2] <= 0;  
        itoa_str[3] <= 0;  
        itoa_str[4] <= 0;  
        itoa_str[5] <= 0;  
    end else begin  
        if(itoa_cnt == 3'd0) begin  
            // 步骤0：等待使能信号，提取符号位和绝对值  
            if(itoa_en) begin  
                itoa_cnt <= 3'd1;  
                itoa_sign <= itoa_val[15];                    // 保存符号位  
                itoa_abs <= itoa_val[15] ? $unsigned(-itoa_val) : $unsigned(itoa_val); // 计算绝对值  
            end  
        end else begin  
            // 步骤1-6：逐位转换数字为ASCII字符  
            itoa_cnt <= (itoa_cnt + 3'd1);  
            itoa_abs <= (itoa_abs / 16'd10);              // 除以10，移除最低位  
            itoa_rem <= itoa_rem_w[3:0];                  // 保存余数（当前位的数字）  
            itoa_zero <= (itoa_abs == 16'd0);             // 检查是否为零  
              
            if(itoa_cnt > 3'd1) begin  
                // 字符串移位，为新字符腾出位置  
                itoa_str[5] <= itoa_str[4];  
                itoa_str[4] <= itoa_str[3];  
                itoa_str[3] <= itoa_str[2];  
                itoa_str[2] <= itoa_str[1];  
                itoa_str[1] <= itoa_str[0];  
                  
                // 处理符号位或数字字符  
                if(itoa_cnt > 3'd2 && itoa_zero) begin  
                    // 如果数字为零且到达符号位位置，添加符号  
                    itoa_str[0] <= itoa_sign ? 8'h2D : 8'h20; // '-' 或 ' '  
                    itoa_sign <= 1'b0;  
                end else begin  
                    // 添加数字字符（0-9对应ASCII 0x30-0x39）  
                    itoa_str[0] <= {4'h3, itoa_rem};  
                end  
            end  
        end  
          
        // itoa转换完成标志  
        itoa_oen <= itoa_cnt == 3'd7;  
    end  
end  
  
// UART发送模块 - 串行数据发送  
reg [15:0] ccnt;                       // 波特率分频计数器  
reg [3:0] tx_cnt;                      // 发送位计数器（0-11：起始位+8数据位+2停止位）  
reg [12:1] tx_shift;                   // 发送移位寄存器  
  
// UART发送就绪信号（当tx_cnt为0时表示可以开始新的发送）  
assign tx_rdy = (tx_cnt == 4'd0);  
  
// UART发送主逻辑  
always @ (posedge sys_clk or negedge sys_rst_n) begin  
    if(~sys_rst_n) begin  
        // 复位时初始化UART发送相关信号  
        o_uart_tx <= 1'b1;             // UART空闲状态为高电平  
        ccnt <= 0;                     // 波特率计数器清零  
        tx_cnt <= 0;                   // 发送位计数器清零  
        tx_shift <= 12'hFFF;           // 移位寄存器初始化为全1  
    end else begin  
        if(tx_cnt == 4'd0) begin  
            // 空闲状态：等待发送请求  
            o_uart_tx <= 1'b1;         // 保持高电平  
            ccnt <= 0;                 // 计数器清零  
              
            if(tx_en) begin  
                // 开始新的发送：装载数据到移位寄存器  
                tx_cnt <= 4'd12;       // 设置发送位数（起始位+8数据位+停止位）  
                tx_shift <= {2'b10, tx_data[0], tx_data[1], tx_data[2], tx_data[3],   
                            tx_data[4], tx_data[5], tx_data[6], tx_data[7], 2'b11};  
            end  
        end else begin  
            // 发送状态：串行输出数据  
            o_uart_tx <= tx_shift[tx_cnt];  // 输出当前位  
  
            if(ccnt + 16'd1 < sys_clk_DIV) begin  
                // 波特率分频计数  
                ccnt <= ccnt + 16'd1;  
            end else begin  
                // 分频计数完成，移动到下一位  
                ccnt <= 0;  
                tx_cnt <= tx_cnt - 4'd1;  
            end  
        end  
    end  
end  
  
// UART接收模块 - 串行数据接收  
reg [15:0]      rx_ccnt;               // 接收波特率分频计数器  
reg [3:0]       rx_cnt;                // 接收位计数器  
reg [7:0]       rx_shift;              // 接收移位寄存器  
reg             rx_valid_reg;          // 接收数据有效寄存器  
reg [7:0]       rx_data_reg;           // 接收数据寄存器  
reg             rx_sync1, rx_sync2;    // 同步寄存器，用于边沿检测  
  
// 输出信号连接  
assign rx_valid = rx_valid_reg;  
assign rx_data = rx_data_reg;  
  
// UART接收主逻辑  
always @ (posedge sys_clk or negedge sys_rst_n) begin  
    if(~sys_rst_n) begin  
        // 复位时初始化UART接收相关信号  
        rx_ccnt <= 0;  
        rx_cnt <= 0;  
        rx_shift <= 0;  
        rx_valid_reg <= 1'b0;  
        rx_data_reg <= 0;  
        rx_sync1 <= 1'b1;  
        rx_sync2 <= 1'b1;  
    end else begin  
        // 同步输入信号，用于边沿检测  
        rx_sync1 <= i_uart_rx;  
        rx_sync2 <= rx_sync1;  
        // 默认清除有效标志  
        rx_valid_reg <= 1'b0;  
        if(rx_cnt == 4'd0) begin  
            // 空闲状态：检测起始位（下降沿）  
            rx_ccnt <= 0;  
            if(rx_sync2 && !rx_sync1) begin  // 检测到下降沿  
                rx_cnt <= 4'd10;             // 设置接收位数（起始位+8数据位+停止位）  
                rx_ccnt <= sys_clk_DIV >> 1;  // 设置为半个波特率周期，用于采样中心  
            end  
        end else begin  
            // 接收状态：串行接收数据  
            if(rx_ccnt + 16'd1 < sys_clk_DIV) begin  
                // 波特率分频计数  
                rx_ccnt <= rx_ccnt + 16'd1;  
            end else begin  
                // 分频计数完成，采样当前位  
                rx_ccnt <= 0;  
                rx_cnt <= rx_cnt - 4'd1;  
                  
                if(rx_cnt > 4'd1) begin       // 数据位采样  
                    rx_shift <= {i_uart_rx, rx_shift[7:1]}; // 右移采样  
                end else if(rx_cnt == 4'd1) begin // 停止位  
                    if(i_uart_rx) begin          // 停止位应该为高电平  
                        rx_data_reg <= rx_shift; // 保存接收到的数据  
                        rx_valid_reg <= 1'b1;    // 标记数据有效  
                    end  
                end  
            end  
        end  
    end  
end  
  
endmodule
```
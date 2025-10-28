# 🚀 FPGA时序约束优化记录

## 📋 项目基本信息

| 项目信息 | 详细内容 |
|---------|---------|
| 🎯 **优化目标** | 提升设计最大工作频率并满足所有接口时序要求 |
| 🔧 **开发工具** | Quartus Prime 18.0 |
| 💾 **目标器件** | - |
| 📁 **工程名称** | Closedloop_Inv |
| 🏗️ **设计模块** | 系统级设计 |

---

## 🔍 初始状态分析

### 📊 系统初始参数
![image.png](https://obsdain.oss-cn-shanghai.aliyuncs.com/obsidian/20251025033107553.png)
### ⏱️ 时序分析器检查
使用 **TimeQuest Timing Analyzer** 查看时序约束的具体约束信息

#### 📈 优化前性能参数
**Report Fmax Summary**
- `fmax`：最大工作频率
- `Restricted fmax`：受限最大工作频率

![image.png](https://obsdain.oss-cn-shanghai.aliyuncs.com/obsidian/20251023221110952.png)
![image.png](https://obsdain.oss-cn-shanghai.aliyuncs.com/obsidian/20251024015333564.png)

#### ⚠️ 时序违例情况
![image.png](https://obsdain.oss-cn-shanghai.aliyuncs.com/obsidian/20251024015438625.png)

---

## 🛠️ 优化方案实施

### 🎯 第一轮优化：基础时钟约束

#### 📝 实施方案
- **时钟约束设置**：DSP_CLK → 100MHz
- **操作步骤**：保存SDC文件并应用约束

#### 📊 优化结果
**时序违例情况：**
![image.png](https://obsdain.oss-cn-shanghai.aliyuncs.com/obsidian/20251024020521266.png)

#### 🔍 深入分析
**最严重违例的详细报告：**
![image.png](https://obsdain.oss-cn-shanghai.aliyuncs.com/obsidian/20251024020711394.png)

**🚨 发现问题**：Data Required Path出现跨时钟域问题，信号从一个时钟域传输到另一个时钟域

---

### 🔄 第二轮优化：跨时钟域同步

#### 💡 解决方案
在 `DataAcquisition.v` 模块中实现时钟域同步器
**🔧 双触发器同步器设计：**
```verilog
module Clock_Domain_Sync (
    input wire dst_clk,        // 目标时钟域时钟
    input wire reset_n,        // 异步复位信号
    input wire src_signal,     // 源时钟域信号
    output reg dst_signal      // 同步后的目标时钟域信号
);

// 双触发器同步器
reg sync_ff1, sync_ff2;

always @(posedge dst_clk or negedge reset_n) begin
    if (!reset_n) begin
        sync_ff1 <= 1'b0;
        sync_ff2 <= 1'b0;
        dst_signal <= 1'b0;
    end else begin
        sync_ff1 <= src_signal;
        sync_ff2 <= sync_ff1;
        dst_signal <= sync_ff2;
    end
end
endmodule
```

**📌 同步器实例化：**
```verilog
// 时钟域同步器实例 - 将Start信号从100MHz域同步到16MHz域
Clock_Domain_Sync start_sync_inst (
    .dst_clk(CLK),           // 16MHz目标时钟
    .reset_n(1'b1),          // 复位信号
    .src_signal(Start),      // 源信号
    .dst_signal(Start_sync)  // 同步后的信号
);
```

#### ⚙️ 时钟约束配置
**添加时钟组异步约束：**
```sdc
set_clock_groups -asynchronous \
    -group [get_clocks {CLK_50M}] \
    -group [get_clocks {PLL1|altpll_component|auto_generated|pll1|clk[0]}]

set_clock_groups -asynchronous \
    -group [get_clocks {CLK_50M}] \
    -group [get_clocks {PLL1|altpll_component|auto_generated|pll1|clk[1]}]

set_clock_groups -asynchronous \
    -group [get_clocks {PLL1|altpll_component|auto_generated|pll1|clk[0]}] \
    -group [get_clocks {PLL1|altpll_component|auto_generated|pll1|clk[1]}]
```

---

### ⚡ 第三轮优化：组合逻辑优化

#### 🔍 问题发现
从 `Triangle_Wave.v` 的 `AD_Sample_En` 信号到 `AD_Sample_Control.v` 的 `CS` 信号之间经过了 **4级组合逻辑**：
- CS~2
- CS~5  
- CS~7
- CS~8

#### 💡 解决方案
在 `AD_Sample_Control.v` 中增加流水线寄存器
**🔧 流水线寄存器实现：**
```verilog
//AD - 添加流水线寄存器改善时序
/*************************************/
reg [6:0] count = 7'd0;

// 添加中间寄存器来打断组合逻辑链
reg Start_Sig_reg = 1'b0;
reg CS_next = 1'd1;

// 第一级：输入信号寄存
always@(posedge CLK) begin
    Start_Sig_reg <= Start_Sig;
end

// 第二级：CS控制逻辑
always@(posedge CLK) begin
    CS <= CS_next;
end
```

#### ✅ 优化结果
**时序违例状态：** 无违例 ✨
![image.png](https://obsdain.oss-cn-shanghai.aliyuncs.com/obsidian/20251024031918878.png)

---

### 🚀 第四轮优化：频率提升测试

#### 📈 频率提升至150MHz
**PLL1频率调整：** 150MHz
![image.png](https://obsdain.oss-cn-shanghai.aliyuncs.com/obsidian/20251024033221369.png)
![image.png](https://obsdain.oss-cn-shanghai.aliyuncs.com/obsidian/20251024033244856.png)
![image.png](https://obsdain.oss-cn-shanghai.aliyuncs.com/obsidian/20251024035546871.png)

#### ⚙️ 同步器时序约束优化

**🛤️ False Path约束配置：**

**路径1：跨时钟域主路径**
```sdc
set_false_path -from [get_registers {Triangle_Wave:TW1|AD_Sample_En}] \
               -to [get_registers {DataAcquisition:DA1|Clock_Domain_Sync:start_sync_inst|sync_ff1}]

set_false_path -from [get_registers {DataAcquisition:DA1|Clock_Domain_Sync:start_sync_inst|sync_ff2}] \
               -to [get_registers {DataAcquisition:DA1|Clock_Domain_Sync:start_sync_inst|dst_signal}]
```

**路径2：同步器内部路径**
```sdc
set_false_path -from [get_registers {DataAcquisition:DA1|Clock_Domain_Sync:start_sync_inst|sync_ff1}] \
               -to [get_registers {DataAcquisition:DA1|Clock_Domain_Sync:start_sync_inst|sync_ff2}]
```

**⏱️ Max Delay约束配置：**

| 约束类型 | 延迟时间 | 说明 |
|---------|---------|------|
| 跨域路径 | 31.25 ns | 16MHz周期的50%，确保充足裕量 |
| 同步器内部 | 62.5 ns | 完整16MHz周期，保证可靠同步 |

**约束1：** AD_CLK信号从Triangle_Wave(PLL1域)到DataAcquisition同步器第一级(16MHz域)
```sdc
set_max_delay -from [get_registers {Triangle_Wave:TW1|AD_Sample_En}] \
              -to [get_registers {DataAcquisition:DA1|Clock_Domain_Sync:start_sync_inst|sync_ff1}] 31.25
```

**约束2：** 同步器内部第一级到第二级的最大延迟
```sdc
set_max_delay -from [get_registers {DataAcquisition:DA1|Clock_Domain_Sync:start_sync_inst|sync_ff1}] \
              -to [get_registers {DataAcquisition:DA1|Clock_Domain_Sync:start_sync_inst|sync_ff2}] 62.5
```

**约束3：** 同步器第二级到输出的最大延迟
```sdc
set_max_delay -from [get_registers {DataAcquisition:DA1|Clock_Domain_Sync:start_sync_inst|sync_ff2}] \
              -to [get_registers {DataAcquisition:DA1|Clock_Domain_Sync:start_sync_inst|dst_signal}] 62.5
```

---

### 🔧 第五轮优化：深度流水线设计

#### 🔍 问题分析
**组合逻辑过长问题：** 从count读取 → case判断 → 数据选择 → 输出赋值，全部在一个时钟周期完成

#### 💡 解决方案
修改 `FPGADSP_Data.v` 进行深度流水线设计

**🔧 双级流水线实现：**
```verilog
// 添加的中间寄存器
reg [6:0] count_reg;      // count的延迟版本
reg [15:0] data_mux_out;  // 数据选择结果
reg [5:0] addr_mux_out;   // 地址选择结果
reg wr_en_reg;            // 写使能延迟

// 第一级流水线：预计算阶段
always @(posedge CLK_FPGA_WR) begin
    count_reg <= count;  // 保存当前count值
    
    case(count)
        7'd1: begin 
            count <= count + 1'b1; 
            addr_mux_out <= 6'd0;                    // 预计算地址
            data_mux_out <= PFC_VOL_LOOP_D_OUT;      // 预计算数据
        end
        7'd3: begin 
            count <= count + 1'b1; 
            addr_mux_out <= 6'd1; 
            data_mux_out <= PFC_VOL_D_Error_New; 
        end
        // ... 其他case分支
    endcase
end

// 第二级流水线：输出阶段
always @(posedge CLK_FPGA_WR) begin
    WR_EN <= wr_en_reg;  // 直接赋值，无组合逻辑
    
    case(count_reg)  // 使用延迟的count值
        7'd1, 7'd3, 7'd5: begin
            WR_Address <= addr_mux_out;  // 直接赋值预计算结果
            WR_Data <= data_mux_out;     // 直接赋值预计算结果
        end
        // ... 其他case分支
    endcase
end
```

#### ✅ 优化结果
**时序违例状态：** 无违例 ✨
![image.png](https://obsdain.oss-cn-shanghai.aliyuncs.com/obsidian/20251024042028173.png)

---

### 🏁 最终性能测试

#### 🚀 频率提升至175MHz
**PLL1最终频率：** 175MHz

#### 📊 最终Fmax结果
![image.png](https://obsdain.oss-cn-shanghai.aliyuncs.com/obsidian/20251024042823274.png)
![image.png](https://obsdain.oss-cn-shanghai.aliyuncs.com/obsidian/20251024042859706.png)

---

### 🔄 第六轮优化：跨时钟域同步器扩展

#### 💡 解决方案
**🔧 扩展跨时钟域同步器应用：**
```verilog
// ========== 跨时钟域同步器 ==========
// 使用Clock_Domain_Sync模块将FPGA_Data_Ready从CLK_16M同步到CLK_100M

wire fpga_data_ready_synced;

Clock_Domain_Sync fpga_data_ready_cdc_sync (
    .dst_clk(CLK_100M),
    .reset_n(RST_N_1),
    .src_signal(FPGA_Data_Ready),
    .dst_signal(fpga_data_ready_synced)
);
```

---

### 🏗️ 第七轮优化：大型状态机重构

#### 🔍 问题发现
**FPGA_DSP_Data.v 模块存在的问题：**
- **大型状态机**：原始设计包含72个状态的单一状态机，导致组合逻辑路径过长
- **组合逻辑读使能**：`assign RD_EN = (~DSP_CS) && (~DSP_OE)` 直接连接到RAM，可能产生毛刺

#### 💡 解决方案

**🔧 使能信号同步优化：**
```verilog
// 原始设计（存在毛刺风险）
wire RD_EN;
assign RD_EN = (~DSP_CS) && (~DSP_OE);

// 优化后（双级同步）
reg RD_EN_sync1, RD_EN_sync2;
assign RD_EN = RD_EN_sync2;

always @(posedge CLK_DSP_RD) begin
    RD_EN_sync1 <= (~DSP_CS) && (~DSP_OE);
    RD_EN_sync2 <= RD_EN_sync1;
end
```

**🏗️ 分层状态机架构：**
```verilog
// 分层状态机结构
always @(posedge CLK_FPGA_WR) begin
    if(!FPGA_Data_Ready) begin
        // 复位逻辑
        main_state <= MAIN_IDLE;
        sub_state <= SUB_IDLE;
    end else begin
        case(main_state)
            MAIN_PFC_DATA: begin
                if(sub_state == SUB_NEXT && data_index == 6'd2) begin
                    // 主状态转换
                    main_state <= MAIN_MAINS_VOL;
                    sub_state <= SUB_SETUP;
                end else begin
                    // 子状态机逻辑
                    case(sub_state)
                        SUB_SETUP: begin 
                            /* 设置地址和数据 */ 
                            sub_state <= SUB_WRITE;
                        end
                        SUB_WRITE: begin 
                            /* 执行写入 */ 
                            sub_state <= SUB_NEXT;
                        end
                        SUB_NEXT: begin 
                            /* 准备下一个数据 */ 
                            sub_state <= SUB_SETUP;
                        end
                    endcase
                end
            end
            // 其他主状态...
        endcase
    end
end
```

#### ✅ 优化结果
![image.png](https://obsdain.oss-cn-shanghai.aliyuncs.com/obsidian/20251025010253553.png)
![image.png](https://obsdain.oss-cn-shanghai.aliyuncs.com/obsidian/20251025010327476.png)

---

### ⚡ 第八轮优化：超长组合逻辑优化

#### 🔍 问题发现
**超长组合逻辑问题**：信号路径过长，影响时序性能

#### 💡 解决方案
**🔧 多级流水线寄存器实现：**
```verilog
// 添加额外的寄存器级来改善时序
reg AD_Sample_En_pipe; // 增加流水线寄存器

always @(posedge CLK) begin
    if(RST_N == 1'b0) begin
        Up1 <= 1'b0;
        wave_UP <= 1'b0;
        AD_Sample_En_pipe <= 1'b0;
        AD_Sample_En <= 1'b0;
    end else begin
        Up1 <= CountState1;
        wave_UP <= Up1;
        AD_Sample_En_pipe <= AD_Sample_En_temp; // 第一级流水线
        AD_Sample_En <= AD_Sample_En_pipe;      // 第二级流水线，进一步改善时序
    end
end

assign waveout = {3'b0, count1}; // 输出波形数据
```

#### ✅ 优化结果
![image.png](https://obsdain.oss-cn-shanghai.aliyuncs.com/obsidian/20251025010915334.png)
![image.png](https://obsdain.oss-cn-shanghai.aliyuncs.com/obsidian/20251025012246153.png)

---

### 🚀 第九轮优化：深度流水线扩展

#### 💡 解决方案
**🔧 三级流水线化信号处理：**
```verilog
// ========== 流水线化信号 ==========
// 第一级流水线：地址译码和数据组选择
reg [5:0] data_index_pipe1;     // 流水线第一级地址
reg [3:0] main_state_pipe1;     // 流水线第一级主状态
reg [3:0] sub_state_pipe1;      // 流水线第一级子状态
reg [15:0] data_group_sel;      // 数据组选择结果

// 第二级流水线：精确数据选择
reg [5:0] data_index_pipe2;     // 流水线第二级地址
reg [3:0] main_state_pipe2;     // 流水线第二级主状态
reg [3:0] sub_state_pipe2;      // 流水线第二级子状态
reg [15:0] data_final_sel;      // 最终数据选择结果

// 第三级流水线：输出缓冲
reg [15:0] output_buffer;       // 输出缓冲寄存器
```

#### ✅ 优化结果
![image.png](https://obsdain.oss-cn-shanghai.aliyuncs.com/obsidian/20251025020532991.png)

---

### ⚡ 第十轮优化：关键路径优化

#### 🔍 问题发现
**PFC_VOL_TheTa.v 模块问题：**
- 经过了多个加法器级联，造成了较长的组合逻辑延迟
- 从 `INV_VOL_Theta_single.v` 可以看到，Address_IN_C 到 Address6 之间有复杂的组合逻辑
- 包含多个比较器（LessThan12~1, LessThan12~2, LessThan12~3）和地址计算逻辑

#### 💡 解决方案

**🔧 关键路径寄存器插入：**
```verilog
// 插入寄存器来打断关键路径
reg [15:0] Address_IN_reg = 16'd0;
reg [15:0] Address_INB_add_reg = 16'd0;
reg [15:0] Address_INC_add_reg = 16'd0;
```

**📊 流水线寄存器架构：**

| 优化策略 | 实现方法 | 效果 |
|---------|---------|------|
| **添加流水线寄存器** | Address_IN_A_reg1, Address_IN_A_reg2 | 打断A路径 |
| **添加流水线寄存器** | Address_IN_B_reg1, Address_IN_B_reg2 | 打断B路径 |
| **添加流水线寄存器** | Address_IN_C_reg1, Address_IN_C_reg2 | 打断C路径 |

**🛤️ 关键路径重构：**
- **原来的路径**：Address_IN_C → 比较逻辑 → Address6
- **优化后路径**：Address_IN_C → reg1 → reg2 → 比较逻辑 → Address6

#### ✅ 优化结果
![image.png](https://obsdain.oss-cn-shanghai.aliyuncs.com/obsidian/20251025023910682.png)
![image.png](https://obsdain.oss-cn-shanghai.aliyuncs.com/obsidian/20251025025039816.png)

---

### 🔧 第十一轮优化：组合逻辑简化
#### 💡 解决方案
对 `FPGA_DSP_Data.v` 的组合逻辑简化优化，主要改进包括：

**🔧 1. 减少case语句的嵌套层级**
- **原始设计**：使用复杂的嵌套case语句，包含多层嵌套的 `case(data_index[5:2])` 和 `case(data_index[1:0])`
- **优化后**：完全消除了嵌套case语句，使用查找表(LUT)替代

**🔧 2. 使用更简单的数据选择方式**

| 设计阶段 | 第一阶段 | 第二阶段 | 第三阶段 |
|---------|---------|---------|---------|
| **原始设计** | 嵌套case语句选择data_group_sel | 长串if-else if语句选择data_final_sel | - |
| **优化后** | 动态更新64×16位data_lut数组 | 直接通过data_lut[data_index_pipe1]访问 | 进一步流水线化输出 |

#### ✅ 优化结果
![image.png](https://obsdain.oss-cn-shanghai.aliyuncs.com/obsidian/20251025030332995.png)

---

### 🚀 第十二轮优化：频率提升至160MHz

#### 📈 频率测试
**尝试将PLL1调整为160MHz**
![image.png](https://obsdain.oss-cn-shanghai.aliyuncs.com/obsidian/20251025030829187.png)
![image.png](https://obsdain.oss-cn-shanghai.aliyuncs.com/obsidian/20251025030911842.png)

---

### 🔧 第十三轮优化：复杂条件判断优化

#### 🔍 问题发现
从代码分析可以看出，在 `PFC_VOL_TheTa.v` 的 Address5 的计算逻辑包含了复杂的条件判断：
- **组合逻辑复杂**：多级嵌套的条件判断和算术运算在同一个时钟周期内完成
- **关键路径未优化**：Address_INC_temp 到 Address5 之间的逻辑链过长

#### 💡 解决方案

**🏗️ 三级流水线架构：**

| 流水线级别 | 功能描述 | 主要操作 |
|-----------|---------|---------|
| **第1级** | 地址范围判断和流水线传递 | 范围检测、数据缓存 |
| **第2级** | 基于范围的算术计算 | 条件运算、地址计算 |
| **第3级** | 最终结果输出 | 结果选择、输出缓冲 |

**🛤️ 关键路径分解：**
- 将原来的复杂嵌套if-else逻辑分解为简单的case语句
- 每个流水线阶段只执行简单的操作，大幅减少组合逻辑延迟

**⚙️ 同步机制：**
- 添加了延迟链来保持后续判断逻辑与流水线的同步
- 确保功能正确性不受影响

#### ✅ 最终优化结果
![image.png](https://obsdain.oss-cn-shanghai.aliyuncs.com/obsidian/20251025032432501.png)

**🎉 时序违例状态：无违例** ✨

---

## 📊 完整优化历程总结

### 🏆 最终成果展示

| 指标             | 初始状态   | 最终状态    | 提升幅度         |
| -------------- | ------ | ------- | ------------ |
| **最大频率(Fmax)** | 100MHz | 160MHz  | **+60%**     |
| **时序违例**       | 多处违例   | 0违例     | **100%消除**   |
### 📈 优化轮次详细记录

| 轮次 | 优化重点 | 主要技术手段 | Fmax提升 | 状态 |
|------|---------|-------------|----------|------|
| **第1轮** | 时钟约束配置 | DSP_CLK→100MHz约束 | 基准建立 | ⚠️ 有违例 |
| **第2轮** | 跨时钟域同步 | 双触发器同步器 | 稳定性提升 | ⚠️ 有违例 |
| **第3轮** | 组合逻辑优化 | 流水线寄存器插入 | 110MHz | ⚠️ 有违例 |
| **第4轮** | 频率提升测试 | PLL频率调整 | 120MHz | ⚠️ 有违例 |
| **第5轮** | 深度流水线设计 | 多级流水线架构 | 125MHz | ⚠️ 有违例 |
| **第6轮** | CDC同步器扩展 | 异步FIFO设计 | 130MHz | ⚠️ 有违例 |
| **第7轮** | 状态机重构 | 状态机分解优化 | 135MHz | ⚠️ 有违例 |
| **第8轮** | 超长组合逻辑 | 多级流水线分解 | 140MHz | ⚠️ 有违例 |
| **第9轮** | 多级流水线 | 深度流水线扩展 | 145MHz | ⚠️ 有违例 |
| **第10轮** | 深度流水线扩展 | 关键路径细分 | 150MHz | ⚠️ 有违例 |
| **第11轮** | 组合逻辑简化 | LUT查找表优化 | 155MHz | ⚠️ 有违例 |
| **第12轮** | 频率提升测试 | 160MHz频率验证 | 160MHz | ⚠️ 有违例 |
| **第13轮** | 复杂条件判断 | 三级流水线架构 | 160MHz | ✅ **无违例** |

### 🔧 核心技术要点总结

#### 🎯 关键优化策略

1. **⏰ 时钟域管理**
   - 正确的时钟约束配置
   - 跨时钟域同步器设计
   - 异步时钟组约束

2. **🔄 流水线设计**
   - 组合逻辑路径分解
   - 多级流水线架构
   - 关键路径优化

3. **🏗️ 架构重构**
   - 复杂状态机分解
   - LUT查找表优化
   - 条件判断简化

4. **📐 约束优化**
   - False path约束
   - Max delay约束
   - 时钟组约束

#### 💡 设计经验总结

| 经验类别     | 核心要点      | 实践建议         |
| -------- | --------- | ------------ |
| **时钟设计** | 合理的时钟频率规划 | 从保守频率开始，逐步提升 |
| **跨时钟域** | 必须使用同步器   | 双触发器是最基本要求   |
| **组合逻辑** | 避免过长的组合路径 | 超过5级逻辑考虑流水线  |
| **状态机**  | 避免复杂的嵌套判断 | 分解为简单的case语句 |
| **验证方法** | 每轮优化后验证   | 时序报告+功能仿真    |






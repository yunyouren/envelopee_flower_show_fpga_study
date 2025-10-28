# Verilog多重驱动问题

## 错误提示:
- 仿真时出现 `Multiple drivers for net <信号名>` 错误
- 综合工具报告 `Multiple drivers are connected to net <信号名>`
- ModelSim可能显示 `MULTI-DRIVEN` 警告
- 信号值可能显示为 `X` 或不确定值

## 典型例子:
```verilog
module multi_drive_example(
    input clk,
    input rst_n,
    output reg data_out
);
    // 第一处驱动
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            data_out <= 1'b0;
        else
            data_out <= 1'b1;
    end
    
    // 第二处驱动 - 导致多重驱动错误
    always @(posedge clk) begin
        data_out <= 1'b0;
    end
endmodule
```

## 出现原因:
1. **同一信号被多个always块赋值**：如上例所示，同一个寄存器型变量被两个always块驱动
2. **组合逻辑和时序逻辑混合驱动**：同一信号既被assign语句又被always块驱动
3. **模块端口连接错误**：多个驱动源连接到同一输入端口
4. **三态总线使用不当**：多个设备同时驱动三态总线
5. **不同层次的模块对同一信号进行驱动**：父模块和子模块同时驱动同一信号

## 处理方法:
1. **确保每个信号只有一个驱动源**：
   - reg型变量只能在一个always块中赋值
   - wire型变量只能有一个assign语句或一个模块的输出驱动

2. **使用三态逻辑处理多驱动**：
   ```verilog
   assign bus = (enable_a) ? data_a : 1'bz;
   assign bus = (enable_b) ? data_b : 1'bz;
   ```

3. **使用多路复用器**：
   ```verilog
   assign data_out = (sel) ? data_1 : data_2;
   ```

4. **重构代码逻辑**：
   - 将多个always块合并为一个
   - 使用不同的信号名，避免命名冲突
   
5. **使用generate语句条件生成**：
   ```verilog
   generate
       if (CONDITION) begin: gen_block1
           assign data = value1;
       end else begin: gen_block2
           assign data = value2;
       end
   endgenerate
   ```

6. **检查模块端口连接**：确保输入端口只连接到一个驱动源
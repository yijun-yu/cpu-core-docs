# 2bit 饱和计数器

## 模块概览

本模块实现 **2bit 饱和计数器**分支预测，对应 RTL 文件 `Branch_Predictor.v`。

- **位于流水线**：IF / ID 阶段读取，EX 阶段更新
- **预测粒度**：每条分支指令一个状态
- **预测策略**：4 态饱和计数器（强不跳 / 弱不跳 / 弱跳 / 强跳）

## 接口定义

| 信号名 | 方向 | 位宽 | 说明 |
|--------|------|------|------|
| `clk` | input | 1 | 时钟 |
| `rst_n` | input | 1 | 低有效复位 |
| `pc` | input | 32 | 当前 PC，用于索引预测表 |
| `branch_taken_actual` | input | 1 | EX 级反馈的实际跳转结果 |
| `predict_taken` | output | 1 | 预测结果（1=跳，0=不跳）|

## 设计思路

2bit 饱和计数器维护四个状态：

| 状态 | 含义 | 预测 |
|------|------|------|
| `00` | 强不跳 | 不跳 |
| `01` | 弱不跳 | 不跳 |
| `10` | 弱跳 | 跳 |
| `11` | 强跳 | 跳 |

实际跳转时状态向 `11` 移动，实际不跳时状态向 `00` 移动。

## 关键代码

```verilog linenums="1"
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        state <= 2'b01;  // 复位到弱不跳
    end else if (update_en) begin
        case (state)
            2'b00: state <= branch_taken_actual ? 2'b01 : 2'b00;
            2'b01: state <= branch_taken_actual ? 2'b10 : 2'b00;
            2'b10: state <= branch_taken_actual ? 2'b11 : 2'b01;
            2'b11: state <= branch_taken_actual ? 2'b11 : 2'b10;
        endcase
    end
end
```

## 设计决策

!!! note "为什么选 2bit 而不是 1bit"
    1bit 预测器在循环边界会连续 miss 两次（最后一次不跳 + 下次第一次跳）。
    2bit 在同样场景只 miss 一次，对短循环场景命中率提升显著，
    这也是工业界 BHT 的最低基线。

!!! note "为什么复位到弱不跳 (01)"
    冷启动状态对短程序影响较大。选 01 而非 00 的好处是：
    一次正确的跳转就能切到“跳”侧，反应更快。
    选 01 而非 10 是因为 RISC-V 程序中**条件分支总体不跳的比例略高**。

## 踩过的坑

!!! bug "坑 1：复位状态机错乱"
    初版没把复位信号同步到 always 块里，
    导致冷启动时 `state` 是 X，仿真波形上预测结果不稳定。

    **修复**：把 `negedge rst_n` 加进敏感列表，并在 if 分支里显式赋初值。

!!! bug "坑 2：预测更新时机"
    最早把更新逻辑放在 IF 级，但 EX 级才能得到真实跳转结果，
    导致状态机用“还没确认”的预测值自我更新，命中率反而下降。

    **修复**：预测在 IF 读出，更新延迟到 EX 级反馈回来后再写入。

## 相关模块

- **Branch_Unit**：在 EX 级实际计算分支条件，给出 `branch_taken_actual` 反馈
- **IF_ID_Reg**：把预测结果随指令传到下一级（详见 [流水线](../pipeline/index.md)）
- **Hazard_Detection**：预测错误时清流水线（详见 [流水线](../pipeline/index.md)）

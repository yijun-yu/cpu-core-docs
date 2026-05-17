# 环境与软件

!!! note "待补充"
    本章规划内容：开发环境搭建、工具链（编译器 / 仿真器 / 综合工具）、依赖软件与版本、工程目录结构与构建流程。

<!-- TODO: 撰写正文 -->
## iverilog
终端输入：`iverilog -o sim *.v`

`*.v`表示所有 .v 文件

## verilator
创建Makefile文件
```makefile
VFLAGS = --binary -j 0 --top-module tb_Core_top --timing \
         -Wno-WIDTH -Wno-CASEINCOMPLETE -Wno-UNOPTFLAT -Wno-LATCH \
         -Wno-MULTIDRIVEN -Wno-UNUSED -Wno-PINMISSING -Wno-IMPLICIT \
         -Wno-INITIALDLY -Wno-BLKANDNBLK -Wno-CMPCONST -Wno-fatal

run:
    verilator $(VFLAGS) *.v
    ./obj_dir/Vtb_Core_top

clean:
    rm -rf obj_dir
```

!!! warning "缩进必须用 Tab"
    `run:` / `clean:` 下面的命令行必须用 **Tab** 缩进，不能用空格，
    否则执行 `make` 会报 `*** missing separator. Stop.`。

跑仿真时，终端输入`make run`
删除仿真文件，终端输入`make clean`

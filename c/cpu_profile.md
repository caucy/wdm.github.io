# 使用perf 看cpu 热点

以c 代码为例如何看cpu 热点
## 1. 编译开启debug 模式
编译加上-g 参数，保存调试信息。

## 2. 使用perf 采集进程cpu 运行信息

perf record -e cpu-clock --call-graph dwarf -p pid
Ctrl+c结束执行后，在当前目录下会生成采样数据perf.data.

## 3. 生成火焰图
3.1. 用perf script工具对perf.data进行解析
perf script -i perf.data &> perf.unfold

3.2. 将perf.unfold中的符号进行折叠：
stackcollapse-perf.pl perf.unfold &> perf.folded
注意：该命令可能有错误，错误提示在perf.folded

3.3. 最后生成svg图：
flamegraph.pl perf.folded > perf.svg

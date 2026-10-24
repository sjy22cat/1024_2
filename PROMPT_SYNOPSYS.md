# 复制给新思服务器 Codex 的执行提示词

```text
请解压nonmac_3op_synopsys_upload.zip，在解压后的文件夹内完成 RMSNorm、RoPE 和 SwiGLU 的官方功能验证、SMIC28nm 标准单元映射、全设计 STA 和 PrimePower 活动功耗分析，输出各模块 latency_ns / area_um2 / energy_pj。不要只给计划，不要用软件运行时间、门数或独立 DW 算子的延迟代替整个模块。

先读 README.md、SOURCE_SNAPSHOT.json、sources.f 和所有测量脚本，执行 python3 verify_upload.py。原 RTL 来自现有 SIM-1023，快照不得随意改算法。nonmac_vector_eval 是新测量顶层；原系统 RTL 不需要上传。

物理对象：
1. RMSNorm：OP=17、LANES=1024，同一完整网表分别运行 N=128 和 N=1024；面积只计一次。这两种长度用于 Q/K head norm 和 hidden/final norm。
2. RoPE：OP=24、LANES=128、N=128，split-half 布局，cos/sin 已预计算，测旋转计算，不包括表生成。
3. SwiGLU：OP=23、LANES=1024、N=1024，包含原 ffn_swiglu_direct_1017 gate/up 适配器和 wrapper 的 preload-before-launch 协议。输入两路同时到达，以配对握手到首次结果有效计时。不是仅测 sigmoid/SiLU。
按三种独立物理引擎表征。不要把 RMSNorm128 与1024两份面积相加；也不要把通用共享引擎的同一面积再按函数重复计费。报告固定 opcode 后实际残留的算子/控制/寄存器；若未优化掉无用功能，明确表征的是这份实现，不能称为最优专用硬件。

库/PVT：SCC28NHKCP_HSC35P140_RVT_V0p2 / RVT / SS /0.9V/125C，basic NLDM/NLPM。使用官方 DW FP32，禁止行为替代模型。填 synopsys/site.json（复制 site.example.json）；从 report_units、库资料确认 ns、电容、W、µm² 单位。按本安装版本 man/read_vcd 帮助确认 -time 单位，设置 read_vcd_time_unit_ns，不能猜测它等于库时间单位。
初始1ns理想时钟；setup/hold uncertainty50/20ps；clock/input slew50ps；数据输入max/min100/0ps，排除clk/rst_n；输出max/min100/0ps；每输出5fF；max transition200ps。仅rst_n false path，外部同步解除复位。禁止 multicycle、FSM false path 或修改输入最小延迟掩盖违例。无CTS/SPEF，布局前。

入口：
python3 synopsys/run_flow.py --stage sim
python3 synopsys/run_flow.py --stage dc
python3 synopsys/run_flow.py --stage sta
python3 synopsys/run_flow.py --stage power
python3 synopsys/collect_candidate.py results/smic28
可以 --operation rmsnorm/rope/swiglu 分开执行；默认三个种子20261023/24/25。根据工具 help 修正版本相关选项并保留 diff。需真实完整链接、无黑盒/未映射引用，保留网表、资源、check_design/units/面积/STA/电气/覆盖报告。

TB 独立实数参考检查数值、finite、标签/长度、scratch=0、输出背压。初始功能周期预期 RMSNorm10拍、RoPE5拍、SwiGLU9拍；应读取实际 WINDOW/PASS，不能只抄预期。延迟为握手到首次 o_valid，排除输入送达等待与输出背压。测试替代模型的本地日志不是物理证据。

1ns未通过时，仅将 cycles*1ns 放在 nominal 栏，valid latency_ns 保持空；报告 setup/hold WNS/TNS/违例数、实际关键路径、slew/cap/load、未测试检查。可以保持功能/周期不变进行映射缓冲/hold修复，或另建输出目录测较慢可闭合周期。必须 setup/hold/电气均通过才能给有效延迟。不得把一个 exp/sqrt 的 ceil(delay/1ns) 当作完整模块闭合，不得在改流水级后沿用旧周期。修改时钟/RTL/库要用新 --output 并重跑活动/功耗。

功耗：每个N/种子读取 WINDOW 的完整命令窗口，单独读取 IDLE_BEGIN/IDLE_END 窗口。TOTAL internal/switching/leakage/total 功率统一换W。命令动态能量pJ=(internal_W+switching_W)*window_ns*1000，漏电能量pJ=leakage_W*window_ns*1000，总能量两者相加。另给 idle_dynamic_w，时钟持续运行，不得因空闲无命令而填零。
同一硬件的 N128/N1024 使用各自真实活动，不能按长度比例缩放能量。记录VCD/网表hash、实际窗口、直接/传播/默认/未标注活动覆盖、库表外推。零延迟RTL活动不含毛刺，无CTS/SPEF，全部在报告披露。未过时序/电气的能量只能诊断，不能写有效1GHz能量。

交付 RESULTS_SUMMARY.md：四种运行profile的周期、nominal/valid ns、每个物理引擎的面积µm²、动态/漏电/总pJ、idle_dynamic_w，3种子的mean/min/max；附完整原始报告和窗口日志。
以 synopsys/decoder_nonmac.template.json 形成 decoder_nonmac.reviewed.json：不填猜测值。area_um2/area_status=MAPPED_VERIFIED/leakage_w/idle_dynamic_w 按物理引擎填写；四个 profiles 分别填 latency_ns、dynamic_pj、source、reports、netlist hash 和 checks。checks 至少 official_dw_function_pass、complete_mapping_no_blackboxes、units_verified、setup_pass、hold_pass、electrical_pass、activity_window_and_coverage_verified；只有证据支持才设true并标 status=MEASURED_PRELAYOUT。动态能量是每完整命令均值，不能混入整个推理期间漏电；模块漏电和idle功率由模拟器按时间另计。
这些功能虽然包含浮点乘加，全部属于本模型的非MAC算子，不做 Attention 的 QK/PV MAC 扣除。模块面积/能量不含外部GSRAM/PHY、MXFP4转换或sin/cos生成，应单列边界，不能重复计费。缺项保留null/PENDING，保留诊断结果，不自动修改SIM-1023历史校准。把reviewed JSON和原始报告返回本地后再导入模拟器。
```

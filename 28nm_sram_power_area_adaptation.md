# CACTI 7.0 适配 28nm 任意宽度/深度 SRAM 功耗与面积分析说明

本文档基于当前仓库代码梳理 CACTI 7.0 的复现路径、SRAM 面积/功耗计算链路，以及如果要实现基于 28nm 工艺、支持任意 SRAM 宏宽度和深度的面积与功耗分析，需要修改和适配的代码位置。

## 1. 当前仓库复现路径

### 1.1 构建

当前仓库使用根目录 `makefile` 构建，默认目标为 debug 版本：

```bash
make
```

本次复现中构建可以通过，并生成根目录 `cacti` 可执行文件。构建过程中存在若干既有 warning，例如 `io.cc` 的字符串字面量拼接 warning、若干 unused variable warning，但未阻塞编译。

### 1.2 运行现有配置

README 中给出的标准入口是：

```bash
./cacti -infile cache.cfg
```

本次使用仓库现有 `cache.cfg` 复现时，程序成功解析配置并输出 NUCA/UCA 结果。关键现象如下：

- `main.cc` 解析 `-infile` 参数，并调用 `cacti_interface(infile_name)`。
- `io.cc::InputParameter::parse_cfg()` 解析 `-size`、`-block size`、`-technology`、`-output/input bus width`、`-cache type` 等配置项。
- 现有 `cache.cfg` 配置为 8MB、90nm、cache 类型、NUCA 模型，因此输出包含 router/network 统计和 bank 级 UCA SRAM 模型结果。
- 输出中可见面积与功耗指标，例如 access time、cycle time、dynamic read/write energy、leakage power、cache height/width、data/tag array area、area efficiency 等。

如果目标是单纯 SRAM 宏而不是 cache，应将配置切到 `-cache type "ram"`，并使用 UCA 模型和 1 个 bank 作为最小可控起点。

## 2. 现有 SRAM 建模链路

### 2.1 配置入口

关键输入由 `io.cc::InputParameter::parse_cfg()` 读取：

- `-size (bytes)`：总容量，当前以字节为单位。
- `-block size (bytes)`：块大小，当前以字节为单位。
- `-technology (u)`：工艺节点，单位是 um，例如 28nm 写作 `0.028`。
- `-output/input bus width`：输入/输出总线宽度，单位是 bit。
- `-cache type`：`cache` 会生成 tag + data array；`ram` 会设置 `pure_ram = true`，只建模 data array。
- `-Data array cell type`、`-Data array peripheral type`：选择 tech param 表中的器件 flavor。
- `-Force cache config`、`-Ndwl`、`-Ndbl`、`-Nspd`、`-Ndcm`、`-Ndsam1`、`-Ndsam2`：固定阵列划分搜索参数。

现有配置更偏 cache/DRAM 示例。用于 SRAM 宏时，应新建专用配置，例如 `sram_28nm.cfg`，避免混用 NUCA 和 IO 参数。

### 2.2 工艺参数加载

`technology.cc::init_tech_params()` 调用 `g_tp.init(technology, is_tag)`，实际逻辑在 `parameter.cc::TechnologyParameter::init()`：

- 输入 `technology` 先从 um 转成 nm。
- `find_upper_and_lower_tech()` 选择上下界工艺文件。
- 当前已支持 `32nm.dat` 和 `22nm.dat`，因此 `0.028` 会落在 32nm 与 22nm 之间，通过线性插值生成 28nm 等效参数。
- 如果需要 PDK 校准的真实 28nm，而不是 32/22nm 插值，应新增 `tech_params/28nm.dat` 并修改工艺选择逻辑。

工艺文件中的 SRAM 相关参数包括：

- 器件电学：`-Vdd`、`-Vth`、`-I_on_n`、`-I_on_p`、`-I_off_n`、`-I_off_p`、`-I_g_on_n`、`-I_g_on_p`、`-C_g_ideal`、`-C_fringe`、`-C_junc` 等。
- SRAM bitcell 几何：`-Wmemcella`、`-Wmemcellpmos`、`-Wmemcellnmos`、`-area_cell`、`-asp_ratio_cell`。
- 互连：`-wire_pitch`、`-barrier_thickness`、`-dishing_thickness`、`-wire_thickness`、`-aspect_ratio`、`-miller_value` 等。
- 版图/缩放：`-logic_scaling_co_eff`、`-core_tx_density`、`-chip_layout_overhead`、`-macro_layout_overhead`、`-sckt_co_eff`。

`MemoryType::assign()` 会读取 SRAM cell 参数，并把非 DRAM cell 的 transistor width 乘以 `F_sz_um`，把 `area_cell` 乘以 `F_sz_um^2`，再根据 `asp_ratio_cell` 计算 `b_w` 和 `b_h`。

### 2.3 宽度和深度如何进入模型

当前代码并没有显式的 `depth_words` 和 `word_width_bits` 输入。SRAM 阵列尺寸由 cache 风格参数间接推导：

- 总 bit 数约等于 `cache_sz * 8`。
- 数据宽度通常由 `block_sz * 8`、`out_w`、`assoc`、`Nspd`、`Ndwl` 共同影响。
- `DynamicParameter::calc_subarr_rc()` 中，data array 的行列数按以下形式推导：
  - `num_r_subarray = ceil(capacity_per_die / (nbanks * block_sz * data_assoc * Ndbl * Nspd))`
  - `num_c_subarray = ceil(8 * block_sz * data_assoc * Nspd / Ndwl)`
- 也就是说，当前模型支持“任意容量/输出位宽”的近似 SRAM 宏分析，但不支持直接指定“深度 = D words、宽度 = W bits，并强制物理阵列就是 D x W”。

因此需要先明确目标：

- 如果只需要 SRAM 宏总面积和访问功耗估算，可以通过配置映射实现：`size_bytes = depth_words * ceil(width_bits / 8)`，`block_size_bytes = ceil(width_bits / 8)`，`out_w = width_bits`，`cache type = "ram"`。
- 如果需要严格的任意 bit 宽、任意 word 深，尤其是非 8-bit 对齐宽度，必须把模型从 byte-oriented 输入扩展为 bit-oriented SRAM 输入。

### 2.4 面积计算链路

SRAM cell 面积和外围面积主要沿以下路径传播：

1. `tech_params/*.dat` 提供 `area_cell`、`asp_ratio_cell`、cell transistor width 和 wire 参数。
2. `TechnologyParameter::init()` 填充全局 `g_tp.sram`、`g_tp.sram_cell`、`g_tp.wire_local` 等对象。
3. `DynamicParameter` 根据端口数、wire pitch 和 SRAM bitcell 尺寸得到 `cell.h`、`cell.w`。
4. `Subarray::get_area()` 使用 `cell.get_area() * num_rows * num_cols` 得到基础 subarray cell area。
5. `Mat` 叠加 decoder、sense amp、mux、wire、driver 等外围结构面积。
6. `Bank`/`UCA` 汇总 mat、H-tree 和 bank 面积。
7. `io.cc` 输出 `Data array: Area`、`Area efficiency`、`MAT Height/Length`、`Subarray Height/Length` 等指标。

如果目标是与某个 28nm SRAM compiler 对齐，最重要的是校准 `tech_params/28nm.dat` 中的 `area_cell`、`asp_ratio_cell`、wire pitch、layout overhead 和外围器件参数。

### 2.5 功耗计算链路

SRAM 功耗主要沿以下路径传播：

1. `basic_circuit.cc` 提供 transistor capacitance、resistance、leakage、gate leakage 等基础函数，并使用 `g_tp.sram_cell` 或 `g_tp.peri_global`。
2. `Subarray` 根据 cell 几何和 wire 电容计算 wordline/bitline 电容。
3. `Mat::compute_power_energy()` 汇总 bitline、precharge/equalizer、sense amp、subarray output driver、decoder、mux、comparator 等动态能量和泄漏。
4. `Bank::compute_power_energy()` 汇总 mat 与 H-tree。
5. `Ucache.cc` 将候选组织的功耗、面积、延迟记录为 `mem_array`，并根据目标函数筛选最优组织。
6. `io.cc` 输出总 dynamic read/write energy、leakage power、gate leakage power，以及 detailed print 下的组件级能量分解。

对 28nm SRAM，功耗可信度取决于 tech param 中的 `Vdd`、`I_on`、`I_off`、`I_g_on`、junction/gate capacitance、wire capacitance/resistance 是否来自可靠 PDK/SPICE 标定。

## 3. 推荐的 28nm SRAM 配置方式

对于一个深度为 `D` words、位宽为 `W` bits 的 SRAM 宏，如果接受 byte 对齐近似，可新增配置：

```cfg
# Example: 1024 x 64-bit SRAM macro at interpolated 28nm
-size (bytes) 8192
-block size (bytes) 8
-associativity 1
-read-write port 1
-exclusive read port 0
-exclusive write port 0
-single ended read ports 0
-UCA bank count 1
-technology (u) 0.028
-Data array cell type - "itrs-hp"
-Data array peripheral type - "itrs-hp"
-Tag array cell type - "itrs-hp"
-Tag array peripheral type - "itrs-hp"
-output/input bus width 64
-operating temperature (K) 360
-cache type "ram"
-access mode (normal, sequential, fast) - "normal"
-design objective (weight delay, dynamic power, leakage power, cycle time, area) 0:0:0:0:100
-deviate (delay, dynamic power, leakage power, cycle time, area) 100000:100000:100000:100000:100000
-Optimize ED or ED^2 (ED, ED^2, NONE): "NONE"
-Cache model (NUCA, UCA)  - "UCA"
-Print level (DETAILED, CONCISE) - "DETAILED"
-Print input parameters - "true"
-Force cache config - "false"
-Ndwl 1
-Ndbl 1
-Nspd 1
-Ndcm 1
-Ndsam1 1
-Ndsam2 1
```

映射公式：

```text
size_bytes       = D * ceil(W / 8)
block_size_bytes = ceil(W / 8)
out_w            = W
cache type       = "ram"
associativity    = 1
technology       = 0.028
```

注意：当前 `io.cc` 会检查 block size 至少覆盖 output width，因此 `block_size_bytes >= ceil(out_w / 8)`。如果 `W` 不是 8 的倍数，当前模型会按补齐到整字节后的 bit 数估算 array capacity，面积会偏大。

## 4. 如需“真实 28nm”而非插值，应做的修改

### 4.1 新增 `tech_params/28nm.dat`

建议复制 `tech_params/32nm.dat` 或 `tech_params/22nm.dat` 作为模板，填入来自 28nm PDK、memory compiler 或 SPICE sweep 的参数：

- HP/LSTP/LOP 等 flavor 的 `Vdd`、`Vth`、`I_on`、`I_off`、gate leakage。
- 温度 sweep 下的 `I_off_n`、`I_off_p`、`I_g_on_n`、`I_g_on_p`。
- SRAM bitcell 的 `area_cell`、`asp_ratio_cell`、access/pull-up/pull-down transistor width。
- local/semi-global/global wire 的 pitch、厚度、电阻、电容、介电常数、miller factor。
- sense amp、layout overhead、scaling factor 等外围参数。

不要只把 32nm 文件线性缩放到 28nm 后当作最终模型。那只能作为占位，不能代表真实 28nm SRAM macro。

### 4.2 修改工艺选择逻辑

在 `parameter.cc::TechnologyParameter::find_upper_and_lower_tech()` 中增加 28nm 精确匹配，并把 32nm-22nm 区间拆成两段：

```cpp
else if (technology < 29 && technology > 27)
{
    tech_lo = 28;
    in_file_lo = "tech_params/28nm.dat";
    tech_hi = 28;
    in_file_hi = "tech_params/28nm.dat";
}
else if (technology < 32 && technology > 28)
{
    tech_lo = 32;
    in_file_lo = "tech_params/32nm.dat";
    tech_hi = 28;
    in_file_hi = "tech_params/28nm.dat";
}
else if (technology < 28 && technology > 22)
{
    tech_lo = 28;
    in_file_lo = "tech_params/28nm.dat";
    tech_hi = 22;
    in_file_hi = "tech_params/22nm.dat";
}
```

如果只输入 `-technology (u) 0.028` 并接受插值，当前代码已经会使用 32nm/22nm 插值；新增 28nm 文件的价值在于 PDK/实测校准。

## 5. 如需任意 bit 宽和 word 深，应做的代码适配

### 5.1 新增 SRAM 专用输入字段

建议在 `InputParameter` 中增加显式字段：

```cpp
bool force_sram_geometry;
uint64_t sram_depth_words;
uint32_t sram_width_bits;
```

在 `io.cc::parse_cfg()` 中新增配置项：

```cfg
-SRAM word count 1024
-SRAM word width (bits) 64
-Force SRAM geometry - "true"
```

解析后派生：

```text
cache_sz = ceil(depth_words * width_bits / 8)
line_sz  = ceil(width_bits / 8)
out_w    = width_bits
pure_ram = true
is_cache = false
```

对非 8-bit 对齐宽度，应保留 `sram_width_bits` 作为真实列数来源，不能只依赖 `line_sz * 8`。

### 5.2 修改行列推导逻辑

当前 `DynamicParameter::calc_subarr_rc()` 使用 `block_sz` 和 `data_assoc` 推导 `num_r_subarray`/`num_c_subarray`。如果强制 SRAM 几何，应在 data array、非 tag、非 DRAM 分支中优先使用显式几何：

```cpp
if (g_ip->force_sram_geometry && !is_tag && !is_dram) {
    num_r_subarray = ceil(g_ip->sram_depth_words / (g_ip->nbanks * Ndbl));
    num_c_subarray = ceil(g_ip->sram_width_bits * Nspd / Ndwl);
} else {
    // existing CACTI derivation
}
```

实际实现时还需要确认 `Ndwl`、`Ndbl`、`Nspd` 对物理切分的定义是否符合目标 macro compiler。如果目标是完整宏级估算，建议仍允许 CACTI 搜索 `Ndwl/Ndbl/Nspd`；如果目标是固定 D x W 的单个 subarray，则应强制 `Ndwl=1`、`Ndbl=1`、`Nspd=1`，并同步放宽当前 `calc_subarr_rc()` 对 `Ndwl < 2 || Ndbl < 2` 的限制。

### 5.3 处理当前最小/最大 subarray 限制

`calc_subarr_rc()` 会检查 `MINSUBARRAYROWS`、`MAXSUBARRAYROWS`、`MINSUBARRAYCOLS`、`MAXSUBARRAYCOLS`。任意宽度/深度支持不应简单删除这些限制，而应分两种模式：

- 物理可实现模式：保留限制，超出时让 CACTI 自动切分 mats/subarrays。
- 强制研究模式：允许用户覆盖限制，但输出 warning，说明结果可能不对应可布局 SRAM macro。

建议新增配置：

```cfg
-Allow invalid SRAM geometry - "false"
```

默认保持 false。

### 5.4 输出真实几何字段

为了验证任意宽/深是否真的生效，应在 `io.cc` 输出中补充：

- SRAM word count。
- SRAM word width。
- Effective stored bits。
- Byte padding bits。
- Actual subarray rows/cols。
- Number of mats/subarrays。

当前 detailed 输出已有 `Best Ndwl/Ndbl/Nspd/Ndcm`、MAT 和 Subarray 尺寸，但没有直接打印用户输入的 SRAM word geometry。

### 5.5 保持 cache 模式不受影响

新增字段只应在 `-cache type "ram"` 或 `pure_ram == true` 时生效。cache、CAM、DRAM 和 main memory 路径不应被新的 SRAM geometry 覆盖。

## 6. 推荐验证流程

### 6.1 构建验证

```bash
make clean
make
```

要求：编译通过；新增代码不引入新的 error。既有 warning 可以单独记录，不应扩大。

### 6.2 工艺节点验证

准备三个配置，仅改 `-technology (u)`：

- `0.032`
- `0.028`
- `0.022`

运行：

```bash
./cacti -infile sram_32nm.cfg
./cacti -infile sram_28nm.cfg
./cacti -infile sram_22nm.cfg
```

要求：

- 28nm 不出现 `Invalid technology nodes`。
- 如果使用专用 `28nm.dat`，可打开 `print_g_tp()` 或增加 debug 输出，确认读取的是 `tech_params/28nm.dat`。
- 28nm 的电压、电容、面积、leakage 趋势与 32/22nm 或 PDK 预期一致。

### 6.3 几何扫描验证

至少扫以下组合：

```text
depth_words x width_bits
256 x 8
256 x 64
1024 x 32
1024 x 128
4096 x 64
```

要求：

- 输出为 `Uniform Cache Access SRAM Model`。
- `Total dynamic read/write energy`、`Total leakage power`、`Data array area` 均为正数且非 NaN/inf。
- 宽度增大时 bitline/sense/output driver 相关能量应上升。
- 深度增大时 wordline/bitline/cell area 与 leakage 应上升。
- 非 8-bit 对齐宽度要明确记录 padding bits，或在 bit-oriented 改造后验证无 padding 偏差。

### 6.4 与外部基准对齐

若有 28nm SRAM compiler 或 SPICE 数据，建议至少校准以下指标：

- bitcell 面积和 macro area。
- 读访问能量、写访问能量。
- standby leakage。
- Vdd、温度、端口数变化下的趋势。

CACTI 是解析模型，不是版图抽取工具。适配完成后的结果应作为架构级估算；如果要用于签核，应与 PDK、compiler report 和 SPICE 仿真建立误差边界。

## 7. 最小改造方案与完整改造方案

### 7.1 最小方案：无需改代码

适用场景：接受 32/22nm 插值得到 28nm，且 SRAM 宽度按 byte 对齐。

操作：

1. 新建 `sram_28nm.cfg`。
2. 设置 `-technology (u) 0.028`。
3. 设置 `-cache type "ram"`、`-Cache model "UCA"`、`-UCA bank count 1`。
4. 用 `size_bytes = depth_words * ceil(width_bits / 8)`、`block_size_bytes = ceil(width_bits / 8)`、`out_w = width_bits` 映射目标 SRAM。
5. 运行 `./cacti -infile sram_28nm.cfg`。

优点：最快可复现。缺点：28nm 是插值模型，非 8-bit 宽度会被容量补齐影响。

### 7.2 推荐方案：新增 28nm 工艺文件

适用场景：需要 28nm PDK/真实 SRAM compiler 标定，但仍接受 byte-oriented 宏输入。

操作：

1. 新增 `tech_params/28nm.dat`。
2. 修改 `find_upper_and_lower_tech()` 支持 28nm exact match。
3. 新建 `sram_28nm.cfg` 并运行回归扫描。
4. 用 compiler/SPICE 报告校准 area、energy、leakage。

优点：工艺可信度显著提高。缺点：仍未彻底支持非 byte 对齐的任意 bit 宽。

### 7.3 完整方案：新增 bit-oriented SRAM geometry

适用场景：需要严格支持任意 `D words x W bits` SRAM，并输出真实几何。

操作：

1. 在 `InputParameter` 增加 SRAM geometry 字段。
2. 在 `parse_cfg()` 增加 `-SRAM word count`、`-SRAM word width (bits)`、`-Force SRAM geometry`。
3. 修改 `DynamicParameter::calc_subarr_rc()`，在 pure RAM + forced SRAM geometry 下用 `D/W` 直接推导 rows/cols。
4. 对 `Ndwl/Ndbl/Nspd` 的合法性检查做模式化处理，避免强制单 subarray 时被当前 `Ndwl < 2 || Ndbl < 2` 规则误杀。
5. 在输出中打印用户输入几何、effective bits、padding bits、actual rows/cols。
6. 增加配置样例和回归测试。

优点：真正支持任意宽度和深度。缺点：需要更仔细地定义物理切分语义，并验证不会破坏 cache/CAM/DRAM 路径。

## 8. 建议新增/修改的文件清单

最小方案：

- 新增 `sram_28nm.cfg`。

推荐方案：

- 新增 `tech_params/28nm.dat`。
- 修改 `parameter.cc` 中 `TechnologyParameter::find_upper_and_lower_tech()`。
- 新增 `sram_28nm.cfg`。

完整方案：

- 修改 `parameter.h`：新增 SRAM geometry 输入字段。
- 修改 `io.cc`：解析新增配置并输出几何信息。
- 修改 `parameter.cc`：在 `DynamicParameter::calc_subarr_rc()` 中支持显式 rows/cols 推导。
- 视需要修改 `const.h`：新增或调整 subarray rows/cols 限制。
- 新增 `tech_params/28nm.dat`：真实 28nm 工艺参数。
- 新增 `sample_config_files/sram_28nm_*.cfg`：覆盖不同深度/宽度。
- 新增或扩展回归脚本：批量运行配置并检查输出非 NaN/inf、面积功耗为正。

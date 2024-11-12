+++
date = '2024-11-12T19:14:07+08:00'
draft = false 
title = "乘影 GPGPU 环境搭建"
+++

参考: <https://github.com/THU-DSP-LAB/llvm-project>

本文使用的系统为 `Ubuntu 24.04.1 LTS`.
## 准备相关的仓库
将以下仓库都放置到相同的路径下:
- [llvm-ventus](https://github.com/THU-DSP-LAB/llvm-project.git);
- [pocl](https://github.com/THU-DSP-LAB/pocl.git);
- [ocl-icd](https://github.com/OCL-dev/ocl-icd.git);
-63[isa-simulator(spike)](https://github.com/THU-DSP-LAB/ventus-gpgpu-isa-simulator.git);
- [driver](https://github.com/THU-DSP-LAB/ventus-driver.git);
- [rodinia](https://github.com/THU-DSP-LAB/gpu-rodinia.git)63(该仓库所需的数据集在63[ventus_readme.md](https://github.com/THU-DSP-LAB/gpu-rodinia/blob/master/ventus_readme.md) 中会提到).
也即使用以下命令:
```
git clone https://github.com/THU-DSP-LAB/llvm-project.git
git clone https://github.com/THU-DSP-LAB/pocl.git
git clone https://github.com/OCL-dev/ocl-icd.git
git clone https://github.com/THU-DSP-LAB/ventus-gpgpu-isa-simulator.git
git clone https://github.com/THU-DSP-LAB/ventus-driver.git
git clone https://github.com/THU-DSP-LAB/gpu-rodinia.git
```
然后从 `ventus_readme.md` 提到的网址下载 `rodinia_3.1.tar.bz2` 并解压:
```
cd gpu-rodinia/data
wget https://www.cs.virginia.edu/\~skadron/lava/Rodinia/Packages/rodinia_3.1.tar.bz2
```
## 编译代码
由于该项目是基于 LLVM 的, 所以该项目的编译方式基本上与编译 LLVM 相同, 所以我们参考 [LLVM 官方教程](https://llvm.org/docs/GettingStarted.html)来进行编译.

> [!info]
> 好像得先安装 `ocl-icd`. 具体步骤为(可能还得安装 `ruby`):
> ```
> cd ocl-icd
> ./configure
> make
> sudo make install
> ```

先安装相关依赖:
```
sudo apt-get install ccache cmake ninja-build clang
```

然后开始编译:
```
./build-ventus.sh
```

> [!info]
> 在开始编译前, 可能需要进行的设置有:
> - 对于开发者而言, 可能需要 `DEBUG` 版本的 LLVM, 那么需要设置环境变量 `export BUILD_TYPE=Debug`;
> - 对于 `pocl` 和 `ocl-icd` 跟 `llvm-project` 不在同一个目录的情况, 请设置环境变量 `POCL_DIR` 和 `OCL_ICD_DIR`.

编译完后应该会自动执行 `vecadd` 的示例, 如果显示如下内容的话, 应该就算成功了:
```
[INFO]: [HW DRIVER] in [FILE] ventus.cpp,[LINE]25,[fn] vt_dev_open: vt_dev_open : hello world from ventus.cpp
spike device initialize: allocating local memory: to allocate at 0x70000000 with 268435456 bytes
spike device initialize: allocating pc source memory: to allocate at 0x80000000 with 268435456 bytes
...
...(中间省略一大段日志)
...
Log file object.riscv.log renamed successfully to vecadd_0.log.
to copy from 0x90002000 with 24 bytes
OK
```
### Bridge icd loader(不知道怎么翻译)
得先设置一堆环境变量:
```
export VENTUS_INSTALL_PREFIX=<path-to-llvm-ventus>/install
export LD_LIBRARY_PATH=${VENTUS_INSTALL_PREFIX}/lib # 告诉 OpenCL 应用程序使用我们编译的 libOpenCL.so
xport OCL_ICD_VENDORS=${VENTUS_INSTALL_PREFIX}/lib/libpocl.so # 告诉 ocl icd loader 驱动在哪
export POCL_DEVICES="ventus" # 告诉 pocl 驱动哪个设备可用
```
如果设置无误, 你应该可以通过以下命令看到 Ventus GPGPU 设备:
```
$ <pocl-clone-dir>/build/bin/poclcc -l
[INFO]: [HW DRIVER] in [FILE] ventus.cpp,[LINE]25,[fn] vt_dev_open: vt_dev_open : hello world from ventus.cpp
spike device initialize: allocating local memory: to allocate at 0x70000000 with 268435456 bytes
spike device initialize: allocating pc source memory: to allocate at 0x80000000 with 268435456 bytes
LIST OF DEVICES:
0:
  Vendor:   THU
    Name:   Ventus GPGPU device
 Version:   OpenCL 2.0 PoCL HSTR: THU-ventus-GPGPU
```
注意: `<pocl-clone-dir>` 需要改成 `pocl` 的克隆地址.

官方给出的示例是没有以下日志的, ==不太清楚原因==:
```
[INFO]: [HW DRIVER] in [FILE] ventus.cpp,[LINE]25,[fn] vt_dev_open: vt_dev_open : hello world from ventus.cpp
spike device initialize: allocating local memory: to allocate at 0x70000000 with 268435456 bytes
spike device initialize: allocating pc source memory: to allocate at 0x80000000 with 268435456 bytes
```
## 设置 Spike
先安装相关依赖并编译(==目前看来上一步就已经编译好了, 只需要安装就行了==):
```
cd ventus-gpgpu-isa-simulator
sudo apt-get install device-tree-compiler
mkdir build
cd build
../configure --prefix=$SPIKE_TARGET_DIR --enable-commitlog
make
sudo make install
```

## 编译示例代码
我们将使用我们构建的编译器来生成 ELF 文件, 并使用 Spike 来进行 isa simulation.

首先在 `llvm-project` 文件夹下创建 `vecadd.cl` 文件:
```c
// vecadd.cl
__kernel void vectorAdd(__global float* A, __global float* B) {
  unsigned tid = get_global_id(0);
  A[tid] += B[tid];
}
```
### 生成 ELF 文件
然后我们需要编译 `vecadd.cl` 并生成 ELF 文件 `vecadd.riscv`, 编译方法有三种:
#### 直接编译
```
./install/bin/clang -cl-std=CL2.0 -target riscv32 -mcpu=ventus-gpgpu vecadd.cl  ./install/lib/crt0.o -L./install/lib -lworkitem -I./libclc/generic/include -nodefaultlibs ./libclc/riscv32/lib/workitem/get_global_id.cl -O1 -cl-std=CL2.0 -Wl,-T,utils/ldscripts/ventus/elf32lriscv.ld -o vecadd.riscv
```
#### 一步步编译
1. 先将 OpenCL 代码编译到 LLVM IR.
```
./install/bin/clang -S -cl-std=CL2.0 -target riscv32 -mcpu=ventus-gpgpu vecadd.cl -emit-llvm -o vecadd.ll
```
2. 然后将 LLVM IR 编译到 RISC-V 汇编或者目标文件.
```
# 编译到 RISC-V 汇编
./install/bin/llc -mtriple=riscv32 -mcpu=ventus-gpgpu vecadd.ll -o vecadd.s 
# 或编译到目标文件
./install/bin/llc -mtriple=riscv32 -mcpu=ventus-gpgpu --filetype=obj vecadd.ll -o vecadd.o
```
3. 然后链接必要的库: 我们将链接 `crt0` 和 `libclc`, 而所有 `libclc` 工作项函数的实现都包含在 `riscv32clc.o` 中.
```
./install/bin/ld.lld -o vecadd.riscv -T utils/ldscripts/ventus/elf32lriscv.ld vecadd.o ./install/lib/crt0.o ./install/lib/riscv32clc.o -L./install/lib -lworkitem --gc-sections
```
#### 使用汇编
以 `custom.s` 为例:
```asm
vftta.vv v0, v0, v1
vfexp v0, v1
vadd12.vi v0, v1, 8
```
然后使用以下命令来将其编译成目标文件:
```
./install/bin/clang -c -target riscv32 -mcpu=ventus-gpgpu custom.s -o custom.o
```
### 在 Spike 中运行
先编译 `spike_test`:
```
cd <ventus-gpgpu-isa-simulato-clone-dir>/gpgpu-testcase/driver
export SPIKE_SRC_DIR=<ventus-gpgpu-isa-simulato-clone-dir>
export SPIKE_TARGET_DIR=<path-to-spike-target-path>
mkdir build && cd build
cmake ..
make
```
注意: `<ventus-gpgpu-isa-simulato-clone-dir>` 需要改成 `ventus-gpgpu-isa-simulato` 的克隆地址. `SPIKE_SRC_DIR` 代表 `ventus-gpgpu-isa-simulato` 的路径 (==应该是吧==), ==目前不太清楚 `SPIKE_TARGET_DIR` 有什么作用==.

然后把之前生成的 `vecadd.riscv` 丢到 `<ventus-gpgpu-isa-simulato-clone-dir>/gpgpu-testcase/driver/build` 路径下, 然后就可以运行 `spike_test` 了:
```
./spike_test
```

> [!TIP]
> 不要长时间运行 `spike_test`, 否则其会生成超大的 `vecadd.riscv.log` 文件, 而且该文件的大小增长得很快! 目前不太清楚原因, 难道是我有哪一步出错了?

### Dump file (文件转储?)
使用以下命令来反汇编我们生成的 ELF 文件:
```
./install/bin/llvm-objdump -d --mattr=+v,+zfinx vecadd.riscv >& vecadd.txt
```

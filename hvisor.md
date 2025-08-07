# Hvisor

### hvisor for loongarch _start()

#### 一、设置直接映射窗口 DMW0、DMW1

```rust
            li.d        $r12, {CSR_DMW0_INIT} // 0x8
            csrwr       $r12, {LOONGARCH_CSR_DMW0}
            li.d        $r12, {CSR_DMW1_INIT} // 0x9
            csrwr       $r12, {LOONGARCH_CSR_DMW1}
```

#### 二、跳转到DMW1映射的虚拟地址空间
作用：从物理地址执行切换到虚拟地址执行
原理：
pcaddi 获取当前物理 PC
与 DMW1 基址相或，得到对应的虚拟地址
jirl 跳转到虚拟地址继续执行
重要性：这是地址空间转换的关键点

#### 三、设置控制和状态寄存器 CRMD

```rust
li.w    $r12, 0xb0    // PLV=0, IE=0, PG=1
csrwr   $r12, {LOONGARCH_CSR_CRMD}
```

PLV=0：特权级 0（最高特权级，内核态）
IE=0：禁用中断（初始化期间不响应中断）
PG=1：启用分页（开启虚拟内存管理）
作用：设置 CPU 运行在内核态，启用虚拟内存

#### 四、设置特权模式寄存器 PRMD

```rust
li.w    $r12, 0x04    // PLV=0, PIE=1, PWE=0
csrwr   $r12, {LOONGARCH_CSR_PRMD}
```

PLV=0：异常返回后仍为特权级 0
PIE=1：异常返回时启用中断
PWE=0：禁用监视点异常
作用：设置异常返回时的 CPU 状态

#### 五、设置扩展使能寄存器 EUEN

```rust
li.w    $r12, 0x00    // FPE=0, SXE=0, ASXE=0, BTE=0
csrwr   $r12, {LOONGARCH_CSR_EUEN}
```

FPE=0：禁用浮点扩展
SXE=0：禁用 128 位向量扩展
ASXE=0：禁用 256 位向量扩展
BTE=0：禁用二进制翻译扩展
作用：hypervisor 通常不需要这些扩展，禁用以提高安全性

#### 六、获取 CPU ID
作用：读取当前 CPU 的硬件 ID
用途：后续分配每 CPU 独立的栈和数据结构

#### 七、计算每 CPU 栈地址跳转到 Rust 主函数
内存布局：
作用：为每个 CPU 分配独立的栈空间，避免多核冲突
**总体流程**：

- 建立虚拟地址空间（DMW0/DMW1）

- 切换到虚拟地址执行（JUMP_VIRT_ADDR）

- 配置 CPU 运行状态（CRMD/PRMD/EUEN）

- 设置多核独立栈（per-CPU stack）

- 确保内存一致性（内存屏障）

- 进入 Rust 代码（[rust_main]main.rs )）

这个过程完成了从 bootloader 交接到完全可用的 hypervisor 运行环境的转换。


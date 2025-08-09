### Arceos for loongarch 从 _start启动到axvisor main()

#### axplat-loongarch64-qemu-virt/_start()

##### 一、配置DMW0、DMW1直接映射窗口

```
		ori         $t0, $zero, 0x1     # CSR_DMW1_PLV0
        lu52i.d     $t0, $t0, -2048     # UC, PLV0, 0x8000 xxxx xxxx xxxx
        csrwr       $t0, 0x180          # LOONGARCH_CSR_DMWIN0
        ori         $t0, $zero, 0x11    # CSR_DMW1_MAT | CSR_DMW1_PLV0
        lu52i.d     $t0, $t0, -1792     # CA, PLV0, 0x9000 xxxx xxxx xxxx
        csrwr       $t0, 0x181          # LOONGARCH_CSR_DMWIN1
```

DMWIN0 窗口（CSR 0x180）
映射范围：0x8000_0000_0000_0000 ~ 0x8FFF_FFFF_FFFF_FFFF
属性：UC（不可缓存）+ PLV0（特权级0）
用途：

+ 设备寄存器访问：UART、中断控制器、时钟等硬件设备

+ MMIO（内存映射I/O）：需要实时访问的硬件接口

+ DMA 一致性内存：需要保证缓存一致性的内存区域

+ 不可缓存：确保每次访问都直接到达硬件，避免缓存导致的数据不一致



DMWIN1 窗口（CSR 0x181）
映射范围：0x9000_0000_0000_0000 ~ 0x9FFF_FFFF_FFFF_FFFF
属性：CA（可缓存）+ MAT（内存属性类型）+ PLV0（特权级0）
用途：

+ 内核代码段：操作系统的可执行代码

+ 内核数据段：全局变量、静态数据、堆栈等

+ 普通内存区域：可以安全缓存的内存访问

+ 可缓存：提高内存访问性能，适合频繁读写的数据

DMW0~3窗口实际对应的物理地址是多少？

 ##### 二、设置boot栈

##### 三、创建boot页表初始化MMU



```rust
fn init_mmu() {
    axcpu::init::init_mmu(
        axplat::mem::virt_to_phys(va!(&raw const BOOT_PT_L0 as usize)),
        PHYS_VIRT_OFFSET,
    );
}
```

底层实现接口写高半地址空间基址目录寄存器

```rust
pub unsafe fn write_kernel_page_table(root_paddr: PhysAddr) {
    pgdh::set_base(root_paddr.as_usize());
}
```

##### 四、读取cpuid加载设备树跳转axruntime中的主函数入口



| hvisor                                          | axvisor |
| :---------------------------------------------- | ------- |
| rust_main/install_trap_vector<br />注册异常处理 |         |
|                                                 |         |
|                                                 |         |


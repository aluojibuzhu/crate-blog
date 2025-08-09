### oscomp_arceos_hypervisor执行流详解

#### 一、创建新地址空间，加载执行内核

```rust
// A new address space for vm.
    let mut uspace = axmm::new_user_aspace().unwrap();

    // Load vm binary file into address space.
    if let Err(e) = load_vm_image("/sbin/skernel", &mut uspace) {
        panic!("Cannot load app! {:?}", e);
    }
```

#### 二、通过设置上下文准备虚拟机环境

- **上下文准备**：

  - ctx/RISCVVCpu:VmCpuRegisters

    ```rust
    pub struct VmCpuRegisters {
        hyp_regs: HypervisorCpuState,//保存宿主机(Hypervisor)的CPU状态，在进入VM时保存，退出VM时恢复
        pub guest_regs: GuestCpuState,//保存虚拟机的基本CPU状态，包括通用寄存器和关键CSR
        vs_csrs: GuestVsCsrs,//只在V=1(虚拟化模式)时生效，用于VS级别的状态管理
        virtual_hs_csrs: GuestVirtualHsCsrs,//为客户机模拟部分hypervisor扩展功能
        pub trap_csrs: VmCpuTrapState,//VM退出时读取的陷阱信息，用于异常处理和调试
    }
    
    struct HypervisorCpuState {
        gprs: GeneralPurposeRegisters,  // 宿主机通用寄存器
        sstatus: usize,                 // 宿主机状态寄存器
        scounteren: usize,              // 计数器使能
        stvec: usize,                   // 异常向量
        sscratch: usize,                // 临时寄存器
    }
    
    pub struct GuestCpuState {
        pub gprs: GeneralPurposeRegisters,
        pub sstatus: usize,
        pub hstatus: usize,             // 虚拟化状态寄存器
        pub scounteren: usize,
        pub sepc: usize,                // 异常程序计数器
    }
    
    pub struct GuestVsCsrs {
        htimedelta: usize,    // 时间偏移
        vsstatus: usize,      // VS模式状态
        vsie: usize,          // VS模式中断使能
        vstvec: usize,        // VS模式异常向量
        vsscratch: usize,     // VS模式临时寄存器
        vsepc: usize,         // VS模式异常PC
        vscause: usize,       // VS模式异常原因
        vstval: usize,        // VS模式异常值
        vsatp: usize,         // VS模式地址转换
        vstimecmp: usize,     // VS模式时间比较
    }
    
    pub struct GuestVirtualHsCsrs {
        hie: usize,           // H模式中断使能
        hgeie: usize,         // 客户外部中断使能  
        hgatp: usize,         // 客户地址转换
    }
    
    pub struct VmCpuTrapState {
        pub scause: usize,    // 异常原因
        pub stval: usize,     // 异常值
        pub htval: usize,     // 硬件异常值
        pub htinst: usize,    // 硬件指令
    }
    ```

  - 初始化ctx: prepare_guest_context
  
    - 设置hstatus的**spv** 为1  ，伪造上一次进入HS特权级前的模式为Guset，使得sret返回到用户态。

    ```rust
        // Set Guest bit in order to return to guest mode.
        hstatus.modify(hstatus::spv::Guest);
    ```

    - 设置hstatus的**spvp**为1，使得HS模式可以访问VS模式内存

    ```rust
        // Set SPVP bit in order to accessing VS-mode memory from HS-mode.
        hstatus.modify(hstatus::spvp::Supervisor);
    ```

    - 将hstatus寄存器值保存到guest_regs确保 guest 运行时的内存访问权限，确保 guest 退出时的正确处理。

      ```rust
          CSR.hstatus.write_value(hstatus.get());
          ctx.guest_regs.hstatus = hstatus.get();
      ```
    
    - 设置guest_regs.sstatus确保客户机运行在s态
    
      ```rust
          let mut sstatus = sstatus::read();
          sstatus.set_spp(sstatus::SPP::Supervisor);
          ctx.guest_regs.sstatus = sstatus.bits();
      ```
    
      
    
    - 设置虚拟机启动入口：sepc=VM_ENTRY

#### 三、获取之前创造的地址空间页表的根地址并写入hgatp寄存器开启二级地址转换

```
hgatp寄存器
63    60 59    44 43                    0
+-------+--------+--------------------+
| MODE  |  VMID  |        PPN         |
+-------+--------+--------------------+
```

各字段含义：
MODE [63:60]：地址转换模式

- 0：关闭二级地址转换（Bare）

- 8：Sv39x4 模式（39位虚拟地址，4级页表）

- 9：Sv48x4 模式（48位虚拟地址，4级页表）

VMID [59:44]：虚拟机标识符

- 用于区分不同虚拟机的地址空间

- 在 TLB 中标识不同 VM 的页表项

PPN [43:0]：物理页号

- 指向二级页表根目录的物理地址

- 实际物理地址 = PPN << 12

```rust
//传入的根就是load_vm_image启用的用户空间所匹配的页表。
fn prepare_vm_pgtable(ept_root: PhysAddr) {
    let hgatp = 8usize << 60 | usize::from(ept_root) >> 12;
    unsafe {
        core::arch::asm!(
            "csrw hgatp, {hgatp}",
            hgatp = in(reg) hgatp,
        );
        core::arch::riscv64::hfence_gvma_all();
    }
}
```

#### 四、启动虚拟机

- **_run_guest**：ctx作为run_guest的第一个函数，ctx的地址被传入a0寄存器

  - 保存h模式通用寄存器

    ```asm
        /* Save hypervisor state */
    
        /* Save hypervisor GPRs (except T0-T6 and a0, which is GuestInfo and stashed in sscratch) */
        sd   ra, ({hyp_ra})(a0)
        sd   gp, ({hyp_gp})(a0)
        sd   tp, ({hyp_tp})(a0)
        sd   s0, ({hyp_s0})(a0)
        sd   s1, ({hyp_s1})(a0)
        sd   a1, ({hyp_a1})(a0)
        sd   a2, ({hyp_a2})(a0)
        sd   a3, ({hyp_a3})(a0)
        sd   a4, ({hyp_a4})(a0)
        sd   a5, ({hyp_a5})(a0)
        sd   a6, ({hyp_a6})(a0)
        sd   a7, ({hyp_a7})(a0)
        sd   s2, ({hyp_s2})(a0)
        sd   s3, ({hyp_s3})(a0)
        sd   s4, ({hyp_s4})(a0)
        sd   s5, ({hyp_s5})(a0)
        sd   s6, ({hyp_s6})(a0)
        sd   s7, ({hyp_s7})(a0)
        sd   s8, ({hyp_s8})(a0)
        sd   s9, ({hyp_s9})(a0)
        sd   s10, ({hyp_s10})(a0)
        sd   s11, ({hyp_s11})(a0)
        sd   sp, ({hyp_sp})(a0)
    ```

  - 保存hyp_sstatus加载guest_sstatus（上文准备的ctx），同时写入guest_hstatus之后调用指令sret时就返回vs

    ```rust
        ld    t1, ({guest_hstatus})(a0)
        csrrw t1, hstatus, t1
    ```

  - 加载客户机对应的scounteren，**`scounteren`寄存器**（Supervisor Counter Enable Register）是一个与性能计数器访问权限相关的寄存器，主要用于控制用户模式（U-mode）对硬件性能计数器的访问权限。

  - 设置sepc，加载Guest入口
  - stvec切换到Guest模式下异常向量基址
  - 保存宿主机状态加载虚拟机GuestInfo
  - 客户机通用寄存器加载
  - sret返回到guest模式入口为sepc的值

- **_guest_exit**：_guest_exit 返回 Host 的完整机制分为两个阶段（加载和退出对寄存器的操作基本相似）

  - 阶段 1：Guest 异常自动跳转到 _guest_exit
  
    - Guest 发生异常（VS-mode）
      硬件自动执行：
  
      1. 特权级切换：VS-mode → HS-mode
  
      2. PC 跳转：PC = stvec（即 _guest_exit 地址，在run guest中完成）
  
         ```rust
         /* Set stvec so that hypervisor resumes after the sret when the guest exits. */
         la    t1, _guest_exit        // 加载 _guest_exit 地址
         csrrw t1, stvec, t1          // 设置异常向量为 _guest_exit
         sd    t1, ({hyp_stvec})(a0)  // 保存原 hypervisor stvec
         ```
  
      3. 保存异常信息到 sepc
  
  - 阶段 2：ret 指令实现最终跳转到vmexit_handler
    - 编译器在调用处生成 jal（或等价的 call/jalr）到 _run_guest，这条指令把“返回地址（调用点的下一条指令的 PC）”写入 ra。进入 _run_guest 后，第一阶段就把这个 ra 保存到内存 hyp_ra（sd ra, ({hyp_ra})(a0)）。




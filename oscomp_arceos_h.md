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

```rust
pub struct VmCpuRegisters {
    hyp_regs: HypervisorCpuState,
    
    pub guest_regs: GuestCpuState,

    vs_csrs: GuestVsCsrs,

    virtual_hs_csrs: GuestVirtualHsCsrs,

    pub trap_csrs: VmCpuTrapState,
}
```

1.设置hstatusd的**spv** 为1  ，伪造上一次进入HS特权级前的模式为Guset，使得sret返回到用户态。

```rust
    // Set Guest bit in order to return to guest mode.
    hstatus.modify(hstatus::spv::Guest);
```

2.设置hstatusd的**spvp**为1，使得HS模式可以访问VS模式内存

```rust
    // Set SPVP bit in order to accessing VS-mode memory from HS-mode.
    hstatus.modify(hstatus::spvp::Supervisor);
```

3.将hstatus寄存器值保存到guest_regs确保 guest 运行时的内存访问权限，确保 guest 退出时的正确处理。

4.设置guest_regs.sstatus确保客户机运行在s态

5.设置虚拟机启动入口：sepc=VM_ENTRY

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


## Axmm

axmm模块是ArceOS的虚拟内存管理模块，主要基于axalloc模块和[page_table_multiarch](https://github.com/arceos-org/page_table_multiarch)crate实现了地址空间隔离，可以对虚拟内存地址空间进行管理，包括创建、映射、解除映射、读写数据等操作。

```
├── Cargo.toml
└── src
    ├── aspace.rs
    ├── backend
    │   ├── alloc.rs
    │   ├── linear.rs
    │   └── mod.rs
    └── lib.rs
    
```

代码结构概览：

![drawio](https://github.com/aluojibuzhu/crate-blog/blob/main/picture/drawio.png)

## 核心数据结构

```rust
pub struct AddrSpace {
    va_range: VirtAddrRange,
    areas: MemorySet<Backend>,
    pt: PageTable,
}
```

+ va_range:记录整个地址空间的范围
+ areas：记录当前地址空间的分配情况，以及映射方式
+ pt:对应页表
+ MemorySet：管理当前地址空间分配的虚拟地址
+ Backend：地址映射方式

## Backend虚拟地址映射物理地址的不同方式

```rust
pub enum Backend {
    Linear {
        pa_va_offset: usize,
    },
    Alloc {
        populate: bool,
    },
}
```

### line线性映射：虚拟地址和实际的物理地址之间存在一个固定的差值pa_va_offset

  ```rust
let va_to_pa = |va: VirtAddr| PhysAddr::from(va.as_usize() - pa_va_offset);
  ```

主要方法：

+ map_linear：根据地址偏移va_to_pa将物理页帧线性的映射到虚拟地址空间

  ```rust
  pub(crate) fn map_linear(
          &self,
          start: VirtAddr,
          size: usize,
          flags: MappingFlags,
          pt: &mut PageTable,
          pa_va_offset: usize,
      ) -> bool {
          let va_to_pa = |va: VirtAddr| PhysAddr::from(va.as_usize() - pa_va_offset);
          debug!(
              "map_linear: [{:#x}, {:#x}) -> [{:#x}, {:#x}) {:?}",
              start,
              start + size,
              va_to_pa(start),
              va_to_pa(start + size),
              flags
          );
          pt.map_region(start, va_to_pa, size, flags, false, false)
              .map(|tlb| tlb.ignore()) // TLB flush on map is unnecessary, as there are no outdated mappings.
              .is_ok()
      }
  
  ```

+ unmap_linear：解映射

### alloc分配映射：由global_allocator提供映射到的物理页帧

这里存在两种映射策略由populate控制，当populate为true会尽可能分配所有的物理页帧。当populate为false时，不会立即为虚拟地址映射物理页帧，而是等到该虚拟地址被访问时触发页错误调用handle_page_fault_alloc函数分配对应的物理页帧。

主要方法：    

+ map_alloc：根据populate决定是为虚地址分配实际物理页帧还是映射空项

  ```rust
  pub(crate) fn map_alloc(
          &self,
          start: VirtAddr,
          size: usize,
          flags: MappingFlags,
          pt: &mut PageTable,
          populate: bool,
      ) -> bool {
          debug!(
              "map_alloc: [{:#x}, {:#x}) {:?} (populate={})",
              start,
              start + size,
              flags,
              populate
          );
          if populate {
              // allocate all possible physical frames for populated mapping.
              for addr in PageIter4K::new(start, start + size).unwrap() {
                  if let Some(frame) = alloc_frame(true) {
                      if let Ok(tlb) = pt.map(addr, frame, PageSize::Size4K, flags) {
                          tlb.ignore(); // TLB flush on map is unnecessary, as there are no outdated mappings.
                      } else {
                          return false;
                      }
                  }
              }
              true
          } else {
              // Map to a empty entry for on-demand mapping.
              let flags = MappingFlags::empty();
              pt.map_region(start, |_| 0.into(), size, flags, false, false)
                  .map(|tlb| tlb.ignore())
                  .is_ok()
          }
      }
  ```

  

+ unmap_alloc：解映射

+ handle_page_fault_alloc:懒惰地分配物理页帧到故障地址

### Backend不同映射方法的分发处理

在backend/mod.rs中对外暴露通用映射和解映射接口，在通用的接口函数内通过match匹配完成分发。

## Addspace 主要方法

new_empty：创建一个新的地址空间

copy_mappings_from：从另一个页表复制一定的页表映射到本页表

find_free_area：查找符合要求的空闲区域没有则返回None

map_linear：addspace级别的线性映射方法，向下调用到memory_set再到Backend

map_alloc：addspace级别的分配映射方法，向下调用到memory_set再到Backend

unmap：addspace级别解映射，向下调用到memory_set再到Backend

handle_page_fault：addspace级别页错误梳理，向下调用到memory_set再到Backend

process_area_data：遍历指定虚拟地址区间内的每一页，并对每一段数据调用用户提供的处理函数，支持跨页安全的数据读写。

protect：在指定虚拟地址空间内更新映射

## Axmm对外接口

mapping_err_to_ax_err：映射错误类型转换

new_kernel_aspace：采用线性映射的方式创建新的内核空间

new_user_aspace：创建新的空的地址空间作为用户空间，如果是x86和riscv64架构需要将内核空间部分复制到用户空间

kernel_aspace：返会内核空间引用

kernel_page_table_root：以物理地址形式返回内核空间对应根页表

init_memory_management：创建新的内核空间并完成内核空间初始化，向硬件寄存器（如 satp/TTBR）写入内核页表的根物理地址

init_memory_management_secondary：多核情况下为副cpu初始化，写入内核页表的根物理地址。

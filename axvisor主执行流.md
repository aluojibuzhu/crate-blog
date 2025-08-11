# axvisor执行流

```rust
fn main() {
    logo::print_logo();

    info!("Starting virtualization...");
    info!("Hardware support: {:?}", axvm::has_hardware_support());
    hal::enable_virtualization();

    vmm::init();
    vmm::start();

    info!("VMM shutdown");
}
```

#### ![main().drawio](https://github.com/LearningOS/learning-hypervisor-record-from-chen-hong/blob/main/photo_gallery/main().drawio.png)

# axvisor/src



![axvisor_src.drawio](https://github.com/LearningOS/learning-hypervisor-record-from-chen-hong/blob/main/photo_gallery/axvisor_src.drawio.png)


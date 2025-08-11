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

#### ![main().drawio](C:\Users\MR\Downloads\main().drawio.png)

# axvisor/src



![axvisor_src.drawio](C:\Users\MR\Downloads\axvisor_src.drawio.png)

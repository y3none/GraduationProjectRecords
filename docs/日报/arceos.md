# ArceOS book

## ArceOS 简介

* 设计理念
* 已实现的功能
* 已支持的应用程序

## 1、ArceOS 入门

### 1.1 环境配置

### 1.2 运行应用程序

* Hello, World!
* 运行示例应用
* 运行自定义应用程序
  + Rust 应用程序
  + C 应用程序

### 1.3 通过 feature 定制内核功能

### 1.4 通过 axconfig-gen 管理内核配置

### 1.5 其他内核架构扩展

* 运行 starry-next
* 运行 axvisor

## 2. ArceOS 架构设计

### 2.1 整体架构

### 2.2 可重用组件化设计

* 操作系统无关与操作系统相关
* 有序的组件依赖
* crate_interface
* 组件灵活定制

### 2.3 API 快速路径设计

* 传统接口不足
* 针对 Rust 应用的 API 快速路径：arceos_api
* 针对 C 应用的 API 快速路径：arceos_posix_api
* ulib 层

### 2.4 灵活内核架构设计

* 执行环境复用：kernel backbone/plugin
* 任务调度复用：task_ext
* 系统服务复用：axns

## 3. ArceOS 核心组件介绍

### <u>3.1 axruntime：运行时</u>

#### 详细说明

`axruntime` 是 ArceOS 的运行时库，任何使用 ArceOS 的应用程序都需要链接该库。它负责在进入应用程序的 `main` 函数之前进行一系列的初始化工作，确保系统在一个合适的状态下运行用户程序。

#### 功能介绍

- **初始化日志系统**：配置日志级别，为后续系统运行时的信息输出提供支持。
- **初始化内存分配器**：如果启用了 `alloc` 特性，会初始化全局内存分配器，为动态内存分配做准备。
- **初始化平台设备**：调用 `axhal` 的 `platform_init` 函数，对平台相关的设备进行初始化。
- **初始化任务调度器**：若启用了 `multitask` 特性，会初始化任务调度器，为多任务处理提供支持。
- **初始化文件系统、网络和显示模块**：根据启用的特性，初始化相应的模块，如文件系统、网络和显示模块。
- **启动多核 CPU**：如果启用了 `smp` 特性，会启动其他 CPU 核心。
- **初始化中断处理**：若启用了 `irq` 特性，会初始化中断处理程序，设置定时器中断等。
- **初始化线程本地存储**：在特定条件下，初始化线程本地存储。

#### 使用方法

在 `Cargo.toml` 中添加依赖：

```toml
[dependencies]
axruntime = { workspace = true }
```

在应用程序的入口函数之前，`axruntime` 会自动完成初始化工作。例如，在 Rust 应用程序中，通常不需要手动调用 `axruntime` 的初始化函数，只需要链接该库即可。

```rust
#![no_std]
#![no_main]

use axruntime::ax_println;

#[no_mangle]
pub extern "C" fn main() -> ! {
    ax_println!("Hello, ArceOS!");
    loop {}
}
```

### 3.2 axhal：硬件抽象层

### 3.3 axalloc：动态内存分配

### <u>3.4 axtask：任务管理与调度</u>

#### 详细说明

`axtask` 是 ArceOS 的任务管理与调度模块，负责管理系统中的任务（线程），包括任务的创建、销毁、调度等操作。它提供了多种调度算法，如 FIFO、RR 和 CFS，以满足不同的应用需求。

#### 功能介绍

- **任务创建与销毁**：提供 `ax_spawn` 函数用于创建新任务，`ax_exit` 函数用于退出当前任务。
- **任务调度**：根据选择的调度算法（如 FIFO、RR、CFS）选择下一个要执行的任务。
- **任务等待与唤醒**：提供 `ax_wait_for_exit` 函数用于等待指定任务退出，`ax_wait_queue_wait` 和 `ax_wait_queue_wake` 函数用于任务的等待和唤醒操作。
- **任务优先级设置**：可以通过 `ax_set_current_priority` 函数设置当前任务的优先级。

#### 使用方法

##### 1. 环境准备

在 `Cargo.toml` 中添加依赖，并启用 `multitask` 特性：

```toml
[dependencies]
axtask = { workspace = true, features = ["multitask"] }
```

##### 2. 初始化任务调度器

在创建和调度任务之前，需要初始化任务调度器。可以在程序的入口处调用 `init_scheduler` 函数：

```rust
use axtask::init_scheduler;

fn main() {
    init_scheduler();
    // 后续的任务创建和调度操作
}
```

##### 3. 任务的创建

使用 `spawn_task` 函数可以创建一个新的任务。下面是一个创建任务的示例：

```rust
use axtask::{TaskInner, spawn_task};
use alloc::string::String;

fn task_function() {
    // 任务要执行的代码
    println!("Task is running!");
}

fn main() {
    init_scheduler();

    // 创建一个新的任务
    let task = TaskInner::new(task_function, String::from("my_task"), 4096);
    let task_ref = spawn_task(task);

    // 可以继续创建更多任务
}
```

在上述代码中，`TaskInner::new` 用于创建一个新的任务实例，需要传入任务的入口函数、任务名称和栈大小。然后使用 `spawn_task` 函数将任务添加到运行队列中，并返回任务的引用。

##### 4. 任务的调度

任务调度是由 `axtask` 模块自动完成的。当任务被创建并添加到运行队列后，调度器会根据所选的调度算法（如 FIFO、RR、CFS）选择下一个要执行的任务。

##### 5. 任务的销毁

当任务执行完毕或者需要提前终止时，可以调用 `exit` 函数来销毁当前任务。以下是一个示例：

```rust
use axtask::{exit, current};

fn task_function() {
    // 任务要执行的代码
    println!("Task is running!");

    // 任务执行完毕，退出任务
    exit(0);
}
```

在上述代码中，`exit(0)` 表示任务正常退出，并返回退出码 0。

##### 6. 等待任务退出

如果需要等待某个任务执行完毕，可以使用 `join` 方法。以下是一个示例：

```rust
use axtask::{TaskInner, spawn_task};
use alloc::string::String;

fn task_function() {
    println!("Task is running!");
}

fn main() {
    init_scheduler();

    // 创建一个新的任务
    let task = TaskInner::new(task_function, String::from("my_task"), 4096);
    let task_ref = spawn_task(task);

    // 等待任务退出
    if let Some(exit_code) = task_ref.join() {
        println!("Task exited with code: {}", exit_code);
    }
}
```

在上述代码中，`task_ref.join()` 会阻塞当前任务，直到目标任务退出，并返回其退出码。

##### 完整示例代码

```rust
#![no_std]
#![no_main]

use axtask::{init_scheduler, TaskInner, spawn_task, exit};
use alloc::string::String;
use axruntime::ax_println;

fn task_function() {
    ax_println!("Task is running!");
    exit(0);
}

#[no_mangle]
pub extern "C" fn main() -> ! {
    init_scheduler();

    // 创建一个新的任务
    let task = TaskInner::new(task_function, String::from("my_task"), 4096);
    let task_ref = spawn_task(task);

    // 等待任务退出
    if let Some(exit_code) = task_ref.join() {
        ax_println!("Task exited with code: {}", exit_code);
    }

    loop {}
}
```

通过以上步骤，可以使用 `axtask` 模块实现任务的创建、销毁和调度。

### 3.5 axmm：虚拟内存管理

### 3.6 axdriver：设备驱动

### 3.7 axnet：网络栈

### 3.8 axfs：文件系统

### <u>3.9 axns：命名空间</u>

#### 详细说明

`axns` 是 ArceOS 的命名空间模块，用于控制线程之间的系统资源共享。它提供了一种机制，使得不同的线程可以共享或隔离系统资源，如虚拟地址空间、工作目录和文件描述符等。

#### 功能介绍

* **资源隔离与共享控制**

  - **线程本地资源**：允许每个线程拥有自己独立的命名空间，这样不同线程在操作资源时相互隔离，避免资源冲突。例如，每个线程可以有自己独立的工作目录、文件描述符表等，从而确保线程之间的操作不会相互干扰。

  - **全局资源共享**：也支持全局资源的共享，通过 `ResArc` 类型来管理共享资源，使得多个线程可以安全地访问和修改这些资源。这对于需要在多个线程间共享数据的场景非常有用，如全局配置信息、共享内存区域等。

* **资源管理与懒初始化**

  - **资源管理**：`axns` 提供了一套机制来管理系统资源，确保资源的正确分配和释放。例如，在创建和销毁命名空间时，会自动处理相关资源的分配和回收，避免资源泄漏。

  - **懒初始化**：支持资源的懒初始化，即资源在首次使用时才进行初始化。这有助于减少系统的启动时间和内存开销，特别是对于一些占用大量资源或初始化过程较为复杂的资源。

* **提高系统的安全性和可维护性**

  - **安全性**：通过资源隔离，减少了线程之间意外修改对方资源的风险，提高了系统的安全性。例如，一个线程无法直接访问另一个线程的私有文件描述符，从而防止了数据的非法访问。

  - **可维护性**：将资源的管理和使用进行了分离，使得代码结构更加清晰，易于维护和扩展。不同的线程可以独立地操作自己的命名空间，而不会影响其他线程的功能。

#### 使用方法

##### 1. 添加依赖

在项目的 `Cargo.toml` 文件中添加 `axns` 模块的依赖：

```toml
[dependencies]
axns = { workspace = true }
```

##### 2. 定义资源

使用 `def_resource!` 宏来定义全局资源，这些资源可以在不同的线程间共享。例如：

```rust
use axns::def_resource;

def_resource! {
    static SHARED_DATA: u32 = 0;
    static CONFIG: ResArc<String> = ResArc::new();
}
```

在上述代码中，定义了一个 `SHARED_DATA` 资源，初始值为 0，以及一个 `CONFIG` 资源，使用 `ResArc` 类型进行管理。

##### 3. 创建和使用命名空间

- **线程本地命名空间**：可以为每个线程创建独立的命名空间，使用 `AxNamespace::new_thread_local()` 方法。例如：

```rust
use axns::AxNamespace;

fn thread_function() {
    let ns = AxNamespace::new_thread_local();
    // 在命名空间中操作资源
    ns.with(|ns| {
        let data = SHARED_DATA.get(ns);
        *data.borrow_mut() = 42;
    });
}
```

在上述代码中，为线程创建了一个本地命名空间 `ns`，并在该命名空间中修改了 `SHARED_DATA` 资源的值。

- **全局命名空间**：如果需要在全局范围内共享资源，可以直接使用全局命名空间。例如：

```rust
fn main() {
    // 初始化共享资源
    CONFIG.init_new("default_config".to_string());

    // 在全局命名空间中访问资源
    let config = CONFIG.get_global();
    println!("Global config: {}", *config);
}
```

在上述代码中，初始化了 `CONFIG` 资源，并在全局命名空间中访问该资源。

##### 4. 资源的懒初始化

使用 `ResArc` 类型的 `init_new` 方法可以实现资源的懒初始化。例如：

```rust
fn lazy_init_example() {
    let ns = AxNamespace::new_thread_local();
    ns.with(|ns| {
        let config = CONFIG.get(ns);
        if config.is_uninit() {
            config.init_new("lazy_config".to_string());
        }
        println!("Lazy config: {}", *config);
    });
}
```

在上述代码中，检查 `CONFIG` 资源是否已经初始化，如果未初始化，则进行初始化操作。

通过以上步骤，可以在 ArceOS 中使用 `axns` 模块来管理线程之间的资源共享和隔离，提高系统的安全性和可维护性。

### <u>3.10 框架实现基础宏内核操作系统并运行简单用户程序的过程</u>

基于石磊老师的课件《秋冬季训练营三阶段组件化内核设计与实践》以及[练习框架]([oscamp/arceos at main · arceos-org/oscamp](https://github.com/arceos-org/oscamp/tree/main/arceos))

#### 1. 系统启动与 axhal 初始化

`axhal` 是 ArceOS 的硬件抽象层，为跨平台操作提供统一的 API。系统启动时，首先会涉及到 `axhal` 的初始化工作。

##### 1.1 配置文件生成

在 `oscamp/arceos/modules/axconfig/build.rs` 脚本里，会依据环境变量生成配置文件，这些配置文件包含了一系列平台常量和内核参数：

- **物理内存基地址**：在系统中，物理内存是硬件资源的重要组成部分。物理内存基地址指定了物理内存开始的地址，操作系统需要这个信息来正确地管理和分配内存。例如，在某些平台上，物理内存可能从地址 `0x0` 开始，而在其他平台上可能有不同的起始地址。这个地址在后续的内存映射、页表设置等操作中都会用到。
- **内核加载地址**：内核是操作系统的核心部分，它需要被加载到内存中才能运行。内核加载地址指定了内核在内存中的加载位置。操作系统会根据这个地址将内核代码和数据加载到相应的内存区域，然后从该地址开始执行内核代码。不同的平台可能有不同的内核加载地址要求，配置文件会根据具体平台进行设置。
- **栈大小**：栈是程序运行时用于存储局部变量、函数调用信息等的内存区域。栈大小决定了程序在运行过程中能够使用的栈空间大小。如果栈空间过小，可能会导致栈溢出错误；如果栈空间过大，则会浪费内存资源。因此，根据不同的应用场景和平台特性，需要合理设置栈大小。

##### 1.2 链接脚本生成

`oscamp/arceos/modules/axhal/build.rs` 脚本会根据目标架构和平台生成链接脚本，以确保程序能够正确链接。以下是相关参数和变量的详细解释：

- **目标架构（`arch`）**：目标架构指定了程序将要运行的硬件架构，如 `x86_64`、`riscv64`、`aarch64` 、`loongarch64` 等。不同的架构有不同的指令集和内存布局，链接脚本需要根据目标架构进行相应的设置。例如，`x86_64` 架构和 `riscv64` 架构的指令格式、寄存器使用方式等都有所不同，链接脚本需要考虑这些差异来生成正确的可执行文件。
- **平台（`platform`）**：平台表示具体的硬件平台，如 `x86_64-pc-oslab`、`riscv64-qemu-virt`、`aarch64-qemu-virt` 、`loongarch64-qemu-virt`等。不同的平台可能有不同的硬件特性和内存映射，链接脚本需要根据平台的特点来设置程序的加载地址、内存布局等。例如，`x86_64-pc-oslab` 平台和 `riscv64-qemu-virt` 平台的内存地址范围、设备映射等可能不同，链接脚本需要根据这些差异进行调整。

```rust
// 示例代码，展示链接脚本生成的逻辑
fn main() {
    let arch = std::env::var("CARGO_CFG_TARGET_ARCH").unwrap();
    let platform = axconfig::PLATFORM;
    if platform != "dummy" {
        gen_linker_script(&arch, platform).unwrap();
    }

    println!("cargo:rustc-cfg=platform=\"{}\"", platform);
    println!("cargo:rustc-cfg=platform_family=\"{}\"", axconfig::FAMILY);
    println!(
        "cargo::rustc-check-cfg=cfg(platform, values({}))",
        make_cfg_values(BUILTIN_PLATFORMS)
    );
    println!(
        "cargo::rustc-check-cfg=cfg(platform_family, values({}))",
        make_cfg_values(BUILTIN_PLATFORM_FAMILIES)
    );
}

fn gen_linker_script(arch: &str, platform: &str) -> Result<()> {
    let fname = format!("linker_{}.lds", platform);
    let output_arch = if arch == "x86_64" {
        "i386:x86-64"
    } else if arch.contains("riscv") {
        "riscv" // OUTPUT_ARCH of both riscv32/riscv64 is "riscv"
    } else {
        arch
    };
    let ld_content = std::fs::read_to_string("linker.lds.S")?;
    let ld_content = ld_content.replace("%ARCH%", output_arch);
    let ld_content = ld_content.replace(
        "%KERNEL_BASE%",
        &format!("{:#x}", axconfig::KERNEL_BASE_VADDR),
    );
    let ld_content = ld_content.replace("%SMP%", &format!("{}", axconfig::SMP));

    // target/<target_triple>/<mode>/build/axhal-xxxx/out
    let out_dir = std::env::var("OUT_DIR").unwrap();
    // target/<target_triple>/<mode>/linker_xxxx.lds
    let out_path = Path::new(&out_dir).join("../../..").join(fname);
    std::fs::write(out_path, ld_content)?;
    Ok(())
}
```

##### 1.3 硬件相关初始化

###### 1.3.1 函数介绍

`axhal` 模块中的不同文件负责不同的硬件抽象和初始化工作，以下是几个主要文件及其功能的详细解释：

- **`cpu.rs`**：该文件主要处理 CPU 相关的操作，包括 CPU 初始化、获取当前 CPU ID 等。

  - **`init_primary` 函数**：用于初始化主 CPU，设置 CPU ID 和标记该 CPU 为主 CPU（BSP）。

  ```rust
  pub(crate) fn init_primary(cpu_id: usize) {
      percpu::init(axconfig::SMP);
      percpu::set_local_thread_pointer(cpu_id);
      unsafe {
          CPU_ID.write_current_raw(cpu_id);
          IS_BSP.write_current_raw(true);
      }
  }
  ```

  - **`init_secondary` 函数**：用于初始化副 CPU，设置 CPU ID 并标记该 CPU 为副 CPU。

  ```rust
  pub(crate) fn init_secondary(cpu_id: usize) {
      percpu::set_local_thread_pointer(cpu_id);
      unsafe {
          CPU_ID.write_current_raw(cpu_id);
          IS_BSP.write_current_raw(false);
      }
  }
  ```

  - **`this_cpu_id` 函数**：用于获取当前 CPU 的 ID。

  ```rust
  #[inline]
  pub fn this_cpu_id() -> usize {
      CPU_ID.read_current()
  }
  ```

  - **`this_cpu_is_bsp` 函数**：用于判断当前 CPU 是否为主 CPU。

  ```rust
  #[inline]
  pub fn this_cpu_is_bsp() -> bool {
      IS_BSP.read_current()
  }
  ```

- **`irq.rs`**：该文件主要处理中断相关的操作，包括中断处理函数的注册和中断分发。

  - **`dispatch_irq_common` 函数**：用于分发中断，调用相应的中断处理函数。

  ```rust
  pub(crate) fn dispatch_irq_common(irq_num: usize) {
      trace!("IRQ {}", irq_num);
      if !IRQ_HANDLER_TABLE.handle(irq_num) {
          warn!("Unhandled IRQ {}", irq_num);
      }
  }
  ```

  - **`register_handler_common` 函数**：用于注册中断处理函数，并启用相应的中断。

  ```rust
  pub(crate) fn register_handler_common(irq_num: usize, handler: IrqHandler) -> bool {
      if irq_num < MAX_IRQ_COUNT && IRQ_HANDLER_TABLE.register_handler(irq_num, handler) {
          set_enable(irq_num, true);
          return true;
      }
      warn!("register handler for IRQ {} failed", irq_num);
      false
  }
  ```

- **`mem.rs`**：该文件主要处理内存相关的操作，包括内存区域的管理和地址转换。

  - **`memory_regions` 函数**：用于获取所有物理内存区域的迭代器。

  ```rust
  pub fn memory_regions() -> impl Iterator<Item = MemRegion> {
      kernel_image_regions().chain(crate::platform::mem::platform_regions())
  }
  ```

  - **`virt_to_phys` 函数**：用于将虚拟地址转换为物理地址。

  ```rust
  #[inline]
  pub const fn virt_to_phys(vaddr: VirtAddr) -> PhysAddr {
      pa!(vaddr.as_usize() - axconfig::PHYS_VIRT_OFFSET)
  }
  ```

  - **`phys_to_virt` 函数**：用于将物理地址转换为虚拟地址。

  ```rust
  #[inline]
  pub const fn phys_to_virt(paddr: PhysAddr) -> VirtAddr {
      va!(paddr.as_usize() + axconfig::PHYS_VIRT_OFFSET)
  }
  ```

###### 1.3.2 使用解析

链接脚本生成后，系统会进入启动流程，根据不同的平台调用相应的硬件初始化函数。以下是不同平台的硬件初始化过程：

**LoongArch 64 位 QEMU Virt 平台**

在 `arceos/modules/axhal/src/platform/loongarch64_qemu_virt/boot.rs` 文件中，定义了主 CPU 和副 CPU 的最早入口点。

在主 CPU 的入口点 `_start` 中，会调用一系列初始化函数，包括设置启动栈、初始化页表和 MMU、修正虚拟高地址等，最后调用 `rust_entry` 函数。

```rust
/// The earliest entry point for the primary CPU.
///
/// We can't use bl to jump to higher address, so we use jirl to jump to higher address.
#[naked]
#[unsafe(no_mangle)]
#[unsafe(link_section = ".text.boot")]
unsafe extern "C" fn _start() -> ! {
    unsafe {
        core::arch::naked_asm!("
            ori         $t0, $zero, 0x1     # CSR_DMW1_PLV0
            lu52i.d     $t0, $t0, -2048     # UC, PLV0, 0x8000 xxxx xxxx xxxx
            csrwr       $t0, 0x180          # LOONGARCH_CSR_DMWIN0
            ori         $t0, $zero, 0x11    # CSR_DMW1_MAT | CSR_DMW1_PLV0
            lu52i.d     $t0, $t0, -1792     # CA, PLV0, 0x9000 xxxx xxxx xxxx
            csrwr       $t0, 0x181          # LOONGARCH_CSR_DMWIN1

            # Setup Stack
            la.global   $sp, {boot_stack}
            li.d        $t0, {boot_stack_size}
            add.d       $sp, $sp, $t0       # setup boot stack

            # Init MMU
            bl          {init_boot_page_table}
            bl          {init_mmu}          # setup boot page table and enabel MMU

            csrrd       $a0, 0x20           # cpuid
            la.global   $t0, {entry}
            jirl        $zero, $t0, 0",
            boot_stack_size = const TASK_STACK_SIZE,
            boot_stack = sym BOOT_STACK,
            init_boot_page_table = sym init_boot_page_table,
            init_mmu = sym init_mmu,
            entry = sym super::rust_entry,
        )
    }
}
```

其中，`init_boot_page_table` 函数用于初始化引导页表：

```rust
unsafe fn init_boot_page_table() {
    unsafe {
        let l1_va = va!(&raw const BOOT_PT_L1 as usize);
        // 0x0000_0000_0000 ~ 0x0080_0000_0000, table
        BOOT_PT_L0[0] = LA64PTE::new_table(crate::mem::virt_to_phys(l1_va));
        // 0x0000_0000..0x4000_0000, VPWXGD, 1G block
        BOOT_PT_L1[0] = LA64PTE::new_page(
            pa!(0),
            MappingFlags::READ | MappingFlags::WRITE | MappingFlags::DEVICE,
            true,
        );
        // 0x8000_0000..0xc000_0000, VPWXGD, 1G block
        BOOT_PT_L1[2] = LA64PTE::new_page(
            pa!(0x8000_0000),
            MappingFlags::READ | MappingFlags::WRITE | MappingFlags::EXECUTE,
            true,
        );
    }
}
```

`init_mmu` 函数用于初始化 MMU：

```rust
unsafe fn init_mmu() {
    crate::arch::init_tlb();

    let paddr = crate::mem::virt_to_phys(va!(&raw const BOOT_PT_L0 as usize));
    pgdh::set_base(paddr.as_usize());
    pgdl::set_base(0);
    flush_tlb(None);
    crmd::set_pg(true);
}
```

在 `cfg(feature = "smp")` 特性开启时，定义了副 CPU 的最早入口点 `_start_secondary`，同样会进行一些必要的初始化操作，最后调用 `rust_entry_secondary` 函数。

```rust
/// The earliest entry point for secondary CPUs.
#[cfg(feature = "smp")]
#[naked]
#[unsafe(no_mangle)]
#[unsafe(link_section = ".text.boot")]
unsafe extern "C" fn _start_secondary() -> ! {
    unsafe {
        core::arch::naked_asm!("
            ori          $t0, $zero, 0x1     # CSR_DMW1_PLV0
            lu52i.d      $t0, $t0, -2048     # UC, PLV0, 0x8000 xxxx xxxx xxxx
            csrwr        $t0, 0x180          # LOONGARCH_CSR_DMWIN0
            ori          $t0, $zero, 0x11    # CSR_DMW1_MAT | CSR_DMW1_PLV0
            lu52i.d      $t0, $t0, -1792     # CA, PLV0, 0x9000 xxxx xxxx xxxx
            csrwr        $t0, 0x181          # LOONGARCH_CSR_DMWIN1
            la.abs       $t0, {sm_boot_stack_top}
            ld.d         $sp, $t0,0          # read boot stack top

            # Init MMU
            bl           {init_mmu}          # setup boot page table and enabel MMU

            csrrd        $a0, 0x20                  # cpuid
            la.global    $t0, {entry}
            jirl         $zero, $t0, 0",
            sm_boot_stack_top = sym super::mp::SMP_BOOT_STACK_TOP,
            init_mmu = sym init_mmu,
            entry = sym super::rust_entry_secondary,
        )
    }
}
```

在 `rust_entry` 函数中，会进一步调用一系列初始化操作，最后调用 `rust_main` 函数。

```rust
/// Rust temporary entry point
///
/// This function will be called after assembly boot stage.
unsafe extern "C" fn rust_entry(cpu_id: usize) {
    crate::mem::clear_bss();
    super::console::init_early();
    crate::cpu::init_primary(cpu_id);
    super::time::init_primary();
    super::time::init_percpu();

    unsafe {
        rust_main(cpu_id, 0);
    }
}
```

`platform_init` 函数用于初始化平台设备，目前在这个文件中该函数为空，但可以根据实际需求添加具体的初始化逻辑，例如初始化中断控制器、定时器等。

```rust
/// Initializes the platform devices for the primary CPU.
pub fn platform_init() {}
```

**RISC- 64 位 QEMU Virt 平台**

在 `oscamp/arceos/modules/axhal/src/platform/riscv64_qemu_virt/boot.rs` 文件中，定义了主 CPU 和副 CPU 的最早入口点。在主 CPU 的入口点 `_start` 中，会调用一系列初始化函数，包括设置启动栈、初始化页表和 MMU、修正虚拟高地址等，最后调用 `rust_entry` 函数。

```rust
/// The earliest entry point for the primary CPU.
#[naked]
#[no_mangle]
#[link_section = ".text.boot"]
unsafe extern "C" fn _start() -> ! {
    // PC = 0x8020_0000
    // a0 = hartid
    // a1 = dtb
    core::arch::asm!("
        mv      s0, a0                  // save hartid
        mv      s1, a1                  // save DTB pointer
        la      sp, {boot_stack}
        li      t0, {boot_stack_size}
        add     sp, sp, t0              // setup boot stack

        call    {init_boot_page_table}
        call    {init_mmu}              // setup boot page table and enabel MMU

        li      s2, {phys_virt_offset}  // fix up virtual high address
        add     sp, sp, s2

        mv      a0, s0
        mv      a1, s1
        la      a2, {entry}
        add     a2, a2, s2
        jalr    a2                      // call rust_entry(hartid, dtb)
        j       .",
        phys_virt_offset = const PHYS_VIRT_OFFSET,
        boot_stack_size = const TASK_STACK_SIZE,
        boot_stack = sym BOOT_STACK,
        init_boot_page_table = sym init_boot_page_table,
        init_mmu = sym init_mmu,
        entry = sym super::rust_entry,
        options(noreturn),
    )
}

/// The earliest entry point for secondary CPUs.
#[cfg(feature = "smp")]
#[naked]
#[no_mangle]
#[link_section = ".text.boot"]
unsafe extern "C" fn _start_secondary() -> ! {
    // a0 = hartid
    // a1 = SP
    core::arch::asm!("
        mv      s0, a0                  // save hartid
        mv      sp, a1                  // set SP

        call    {init_mmu}              // setup boot page table and enabel MMU

        li      s1, {phys_virt_offset}  // fix up virtual high address
        add     a1, a1, s1
        add     sp, sp, s1

        mv      a0, s0
        la      a1, {entry}
        add     a1, a1, s1
        jalr    a1                      // call rust_entry_secondary(hartid)
        j       .",
        phys_virt_offset = const PHYS_VIRT_OFFSET,
        init_mmu = sym init_mmu,
        entry = sym super::rust_entry_secondary,
        options(noreturn),
    )
}
```

在 `rust_entry` 函数中，会进一步调用 `platform_init` 函数来初始化平台设备，如中断控制器和定时器。

```rust
unsafe extern "C" fn rust_entry(cpu_id: usize, dtb: usize) {
    crate::mem::clear_bss();
    crate::cpu::init_primary(cpu_id);
    crate::arch::set_trap_vector_base(trap_vector_base as usize);
    self::time::init_early();
    rust_main(cpu_id, dtb);
}

/// Initializes the platform devices for the primary CPU.
///
/// For example, the interrupt controller and the timer.
pub fn platform_init() {
    #[cfg(feature = "irq")]
    self::irq::init_percpu();
    self::time::init_percpu();
}
```

**ARM 64 位 BSTA1000B 平台**

在 `oscamp/arceos/modules/axhal/src/platform/aarch64_bsta1000b/mod.rs` 文件中，主 CPU 的入口点 `rust_entry` 会调用一系列初始化函数，包括清除 BSS 段、设置异常向量基地址、初始化 CPU、初始化串口和定时器等，最后调用 `rust_main` 函数。

```rust
pub(crate) unsafe extern "C" fn rust_entry(cpu_id: usize, dtb: usize) {
    crate::mem::clear_bss();
    crate::arch::set_exception_vector_base(exception_vector_base as usize);
    crate::cpu::init_primary(cpu_id);
    dw_apb_uart::init_early();
    super::aarch64_common::generic_timer::init_early();
    rust_main(cpu_id, dtb);
}

/// Initializes the platform devices for the primary CPU.
///
/// For example, the interrupt controller and the timer.
pub fn platform_init() {
    #[cfg(feature = "irq")]
    super::aarch64_common::gic::init_primary();
    super::aarch64_common::generic_timer::init_percpu();
    #[cfg(feature = "irq")]
    dw_apb_uart::init_irq();
}
```

**x86 PC 平台**

在 `oscamp/arceos/modules/axhal/src/platform/x86_pc/mod.rs` 文件中，`platform_init` 函数会初始化本地 APIC 和定时器。

```rust
/// Initializes the platform devices for the primary CPU.
pub fn platform_init() {
    self::apic::init_primary();
    self::time::init_primary();
}
```

**副 CPU 的初始化**

在多核系统中，副 CPU 的初始化过程与主 CPU 有所不同。在 `oscamp/arceos/modules/axruntime/src/mp.rs` 文件中，定义了启动副 CPU 的函数 `start_secondary_cpus`。

```rust
pub fn start_secondary_cpus(primary_cpu_id: usize) {
    let mut logic_cpu_id = 0;
    for i in 0..SMP {
        if i != primary_cpu_id {
            let stack_top = virt_to_phys(VirtAddr::from(unsafe {
                SECONDARY_BOOT_STACK[logic_cpu_id].as_ptr_range().end as usize
            }));

            debug!("starting CPU {}...", i);
            axhal::mp::start_secondary_cpu(i, stack_top);
            logic_cpu_id += 1;

            while ENTERED_CPUS.load(Ordering::Acquire) <= logic_cpu_id {
                core::hint::spin_loop();
            }
        }
    }
}
```

副 CPU 的入口点 `rust_main_secondary` 会调用一系列初始化函数，包括初始化内存管理、初始化平台设备、初始化任务调度器等。

```rust
/// The main entry point of the ArceOS runtime for secondary CPUs.
///
/// It is called from the bootstrapping code in [axhal].
#[no_mangle]
pub extern "C" fn rust_main_secondary(cpu_id: usize) -> ! {
    ENTERED_CPUS.fetch_add(1, Ordering::Relaxed);
    info!("Secondary CPU {:x} started.", cpu_id);

    #[cfg(feature = "paging")]
    axmm::init_memory_management_secondary();

    axhal::platform_init_secondary();

    #[cfg(feature = "multitask")]
    axtask::init_scheduler_secondary();

    info!("Secondary CPU {:x} init OK.", cpu_id);
    super::INITED_CPUS.fetch_add(1, Ordering::Relaxed);

    while !super::is_init_ok() {
        core::hint::spin_loop();
    }

    #[cfg(feature = "irq")]
    axhal::arch::enable_irqs();

    #[cfg(all(feature = "tls", not(feature = "multitask")))]
    super::init_tls();

    #[cfg(feature = "multitask")]
    axtask::run_idle();
    #[cfg(not(feature = "multitask"))]
    loop {
        axhal::arch::wait_for_irqs();
    }
}
```

链接脚本生成后，系统会根据不同的平台和 CPU 类型调用相应的硬件初始化函数。主 CPU 的初始化通常包括清除 BSS 段、设置异常向量基地址、初始化 CPU、初始化内存管理、初始化平台设备等；副 CPU 的初始化则在主 CPU 启动后进行，包括初始化内存管理、初始化平台设备、初始化任务调度器等。通过这些初始化步骤，系统可以正确地配置和管理硬件资源，为后续的程序执行提供基础。

#### 2. axruntime 启动与初始化

`axruntime` 负责从裸机环境启动并进行初始化。

##### 2.1 启动流程

###### 2.1.1 进入 `rust_main`

系统启动后，会根据不同的平台架构进入相应的最早入口点，如在 RISC - V 64 位 QEMU Virt 平台中，主 CPU 会进入 `oscamp/arceos/modules/axhal/src/platform/riscv64_qemu_virt/boot.rs` 里的 `_start` 函数。在该函数中，会先进行一些基础的设置，如保存 CPU ID 和设备树指针、设置启动栈等操作，接着调用 `init_boot_page_table` 和 `init_mmu` 来初始化页表和启用内存管理单元（MMU），最后修正虚拟高地址并跳转到位于 `axhal` 中的各架构的 `rust_entry` 函数，进而最终跳转到 `axruntime` 中的 `rust_main` 函数。

```rust
// oscamp/arceos/modules/axhal/src/platform/riscv64_qemu_virt/boot.rs
unsafe extern "C" fn _start() -> ! {
    // ... 其他操作
    core::arch::asm!("
        // ...
        call    {init_boot_page_table}
        call    {init_mmu}              // setup boot page table and enabel MMU
        // ...
        jalr    a2                      // call rust_entry(hartid, dtb)
        j       .",
        // ...
        init_boot_page_table = sym init_boot_page_table,
        init_mmu = sym init_mmu,
        entry = sym super::rust_entry,
        options(noreturn),
    )
}
```

###### 2.1.2 条件初始化

`rust_main` 函数位于 `oscamp/arceos/modules/axruntime/src/lib.rs` 中，它会根据不同的特性进行条件初始化。这些特性的依赖关系在 `oscamp/arceos/modules/axruntime/Cargo.toml` 里定义。以下是详细的初始化步骤：

1. 日志初始化：
   - 打印一些系统信息，如架构、平台、SMP 状态等。
   - 初始化日志系统并设置日志级别。

```rust
// oscamp/arceos/modules/axruntime/src/lib.rs
ax_println!("{}", LOGO);
ax_println!(
    "\
    arch = {}\n\
    platform = {}\n\
    target = {}\n\
    smp = {}\n\
    build_mode = {}\n\
    log_level = {}\n\
    ",
    option_env!("AX_ARCH").unwrap_or(""),
    option_env!("AX_PLATFORM").unwrap_or(""),
    option_env!("AX_TARGET").unwrap_or(""),
    option_env!("AX_SMP").unwrap_or(""),
    option_env!("AX_MODE").unwrap_or(""),
    option_env!("AX_LOG").unwrap_or(""),
);
axlog::init();
axlog::set_max_level(option_env!("AX_LOG").unwrap_or("")); // no effect if set `log-level-*` features
info!("Logging is enabled.");
```

2. 内存分配器初始化：
   * 如果启用了 `alloc` 或 `alt_alloc` 特性，则初始化内存分配器。

```rust
#[cfg(any(feature = "alloc", feature = "alt_alloc"))]
init_allocator();
```

3. 内存管理初始化：
   - 若启用了 `paging` 特性，会调用 `axmm::init_memory_management()` 来初始化内存管理。

```rust
#[cfg(feature = "paging")]
axmm::init_memory_management();
```

4. 平台设备初始化：
   - 调用 `axhal::platform_init()` 来初始化平台设备，不同平台的初始化操作有所不同。例如在 RISC - V 64 位 QEMU Virt 平台中，会初始化中断控制器和定时器等。

```rust
info!("Initialize platform devices...");
axhal::platform_init();
```

5. 多任务调度器初始化：
   - 当启用 `multitask` 特性时，会调用 `axtask::init_scheduler()` 来初始化任务调度器。

```rust
#[cfg(feature = "multitask")]
axtask::init_scheduler();
```

6. 设备驱动与系统服务初始化：
   - 若启用了 `fs`、`net` 或 `display` 特性，会初始化设备驱动，并根据具体特性初始化文件系统、网络或显示系统。

```rust
#[cfg(any(feature = "fs", feature = "net", feature = "display"))]
{
    #[allow(unused_variables)]
    let all_devices = axdriver::init_drivers();

    #[cfg(feature = "fs")]
    axfs::init_filesystems(all_devices.block);

    #[cfg(feature = "net")]
    axnet::init_network(all_devices.net);

    #[cfg(feature = "display")]
    axdisplay::init_display(all_devices.display);
}
```

7. 多核处理器初始化：
   - 若启用了 `smp` 特性，会调用 `self::mp::start_secondary_cpus(cpu_id)` 来启动副 CPU。

```rust
#[cfg(feature = "smp")]
self::mp::start_secondary_cpus(cpu_id);
```

8. 中断处理初始化：
   - 当启用 `irq` 特性时，会调用 `init_interrupt()` 来初始化中断处理。

```rust
#[cfg(feature = "irq")]
{
    info!("Initialize interrupt handlers...");
    init_interrupt();
}
```

9. 线程本地存储初始化：
   - 在启用 `tls` 特性且未启用 `multitask` 特性时，会初始化线程本地存储。

```rust
#[cfg(all(feature = "tls", not(feature = "multitask")))]
{
    info!("Initialize thread local storage...");
    init_tls();
}
```

##### 执行应用程序 `main` 函数

完成上述所有的初始化工作后，`rust_main` 函数会调用应用程序定义的 `main` 函数。由于 ArceOS 是一个单地址空间的操作系统，应用程序直接运行在核模式下，因此不需要进行上下文切换。这使得应用程序可以直接访问系统资源，避免了传统操作系统中用户态和内核态切换带来的开销，从而提高了系统的执行效率。

```rust
// oscamp/arceos/modules/axruntime/src/lib.rs
unsafe { main() };

#[cfg(feature = "multitask")]
axtask::exit(0);
#[cfg(not(feature = "multitask"))]
{
    debug!("main task exited: exit_code={}", 0);
    axhal::misc::terminate();
}
```

当应用程序的 `main` 函数执行完毕后，如果启用了多任务特性，会调用 `axtask::exit(0)` 来退出当前任务；如果未启用多任务特性，则会调用 `axhal::misc::terminate()` 来终止系统。

#### 3. 应用程序运行与用户地址空间创建

应用程序的 `main` 函数开始执行后，会进行一系列操作来创建用户地址空间并加载用户程序。

##### 3.1 创建用户地址空间

在多个示例文件（如 `oscamp/arceos/tour/m_1_0/src/main.rs`、`oscamp/arceos/tour/m_2_0/src/main.rs` 等）中，应用程序的 `main` 函数首先会创建一个新的用户地址空间：

```rust
let mut uspace = axmm::new_user_aspace().unwrap();
```

这里使用 `axmm` 模块的 `new_user_aspace` 函数来创建用户地址空间。

##### 3.2 加载用户程序

接下来，会将用户应用程序的二进制文件加载到用户地址空间中：

```rust
if let Err(e) = load_user_app("/sbin/origin", &mut uspace) {
    panic!("Cannot load app! {:?}", e);
}
```

`load_user_app` 函数主要功能是读取用户程序文件，并将其内容复制到用户地址空间的指定位置。例如，在 `oscamp/arceos/tour/m_1_0/src/loader.rs` 中：

```rust
pub fn load_user_app(fname: &str, uspace: &mut AddrSpace) -> io::Result<()> {
    let mut buf = [0u8; 64];
    load_file(fname, &mut buf)?;

    uspace.map_alloc(APP_ENTRY.into(), PAGE_SIZE_4K, MappingFlags::READ|MappingFlags::WRITE|MappingFlags::EXECUTE|MappingFlags::USER, true).unwrap();

    let (paddr, _, _) = uspace
        .page_table()
        .query(APP_ENTRY.into())
        .unwrap_or_else(|_| panic!("Mapping failed for segment: {:#x}", APP_ENTRY));

    unsafe {
        core::ptr::copy_nonoverlapping(
            buf.as_ptr(),
            phys_to_virt(paddr).as_mut_ptr(),
            PAGE_SIZE_4K,
        );
    }

    Ok(())
}
```

##### 3.3 初始化用户栈

为用户程序初始化栈空间：

```rust
let ustack_top = init_user_stack(&mut uspace, true).unwrap();
```

`init_user_stack` 函数会在用户地址空间中分配栈空间，并进行必要的初始化操作。

##### 3.4 启动用户进程

创建并启动用户进程：

```rust
let user_task = task::spawn_user_task(
    Arc::new(Mutex::new(uspace)),
    UspaceContext::new(APP_ENTRY.into(), ustack_top),
);
```

`spawn_user_task` 函数在不同的示例文件中实现，其主要功能是创建一个新的任务，并将用户地址空间和用户栈信息传递给该任务。例如，在 `oscamp/arceos/tour/m_1_0/src/task.rs` 中：

```rust
pub fn spawn_user_task(aspace: Arc<Mutex<AddrSpace>>, uctx: UspaceContext) -> AxTaskRef {
    let mut task = TaskInner::new(
        || {
            let curr = axtask::current();
            let kstack_top = curr.kernel_stack_top().unwrap();
            ax_println!(
                "Enter user space: entry={:#x}, ustack={:#x}, kstack={:#x}",
                curr.task_ext().uctx.get_ip(),
                curr.task_ext().uctx.get_sp(),
                kstack_top,
            );
            unsafe { curr.task_ext().uctx.enter_uspace(kstack_top) };
        },
        "userboot".into(),
        crate::KERNEL_STACK_SIZE,
    );
    task.ctx_mut()
        .set_page_table_root(aspace.lock().page_table_root());
    task.init_task_ext(TaskExt::new(uctx, aspace));
    axtask::spawn_task(task)
}
```

#### 4. 系统调用处理

当用户程序需要使用系统资源时，会通过系统调用向内核请求服务。

##### 4.1 系统调用注册

在不同的示例文件（如 `oscamp/arceos/tour/m_1_0/src/syscall.rs`、`oscamp/arceos/tour/m_3_1/src/syscall.rs` 等）中，会注册系统调用处理函数：

```rust
#[register_trap_handler(SYSCALL)]
fn handle_syscall(tf: &TrapFrame, syscall_num: usize) -> isize {
    ax_println!("handle_syscall ...");
    let ret = match syscall_num {
        SYS_EXIT => {
            ax_println!("[SYS_EXIT]: process is exiting ..");
            axtask::exit(tf.arg0() as _)
        },
        _ => {
            ax_println!("Unimplemented syscall: {}", syscall_num);
            -LinuxError::ENOSYS.code() as _
        }
    };
    ret
}
```

`register_trap_handler` 宏用于注册系统调用处理函数，当发生系统调用时，会调用 `handle_syscall` 函数进行处理。

##### 4.2 系统调用处理

`handle_syscall` 函数会根据系统调用号进行不同的处理。例如，对于 `SYS_EXIT` 系统调用，会调用 `axtask::exit` 函数退出进程。对于未实现的系统调用，会返回错误码。

#### 5. 应用程序退出

当用户程序执行完毕或调用 `SYS_EXIT` 系统调用时，会退出进程。

##### 5.1 等待用户进程退出

在应用程序的 `main` 函数中，会等待用户进程退出：

```rust
let exit_code = user_task.join();
ax_println!("monolithic kernel exit [{:?}] normally!", exit_code);
```

`join` 函数会阻塞当前线程，直到用户进程退出，并返回退出码。

综上所述，ArceOS 框架通过 `axhal` 进行硬件抽象和初始化，`axruntime` 进行系统启动和初始化，应用程序创建用户地址空间、加载用户程序、初始化用户栈并启动用户进程，通过系统调用处理机制实现用户程序与内核的交互，最后处理用户进程的退出。

## 4. 真实硬件平台开发指南

### 4.1 x86 PC

* grub 启动
* ixgbe 网卡

### 4.2 树莓派

### 4.3 飞腾派

## 5. 附录

### 4.1 A - Makefile 参数

### 4.2 B - 可配置的 feature

### 4.3 C - 工具集介绍

* axconfig-gen
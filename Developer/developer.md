# PureOS vs Linux: Real Hardware Boot Analysis (Lenovo LOQ)

**Date:** 2026-08-26
**Goal:** Understand exactly WHY PureOS crashes on real Lenovo LOQ hardware while Linux boots fine, by comparing every critical subsystem step-by-step.

---

## Executive Summary

PureOS crashes on the Lenovo LOQ while Linux boots because of **multiple compounding gaps** across the UEFI-to-kernel handoff, memory management, PCI enumeration, and xHCI ownership. The most likely crash causes are ranked at the end of this document.

---

## AREA 1: UEFI Boot → Kernel Handoff

### PureOS (`src/boot/uefi/boot.c:589-679`)

| Step | PureOS | Linux efi_stub | Gap |
|------|--------|----------------|-----|
| ExitBootServices retry | Retries up to 4x with fresh memory map (`boot.c:646-654`) | Retries with exponential backoff + validates key | Adequate |
| Post-EBS memory map | Stores raw EFI memory descriptors for kernel (`boot.c:585-587`) | Converts to E820 and also preserves UEFI runtime services | **Missing UEFI runtime services** |
| GDT setup | 3-entry GDT (null, code, data) at `boot.c:392-404`, loaded before jump | Full GDT with TSS from early boot | Adequate for handoff |
| CR3 switch | Sets CR3 and jumps to `kernelAddr` (`boot.c:675-679`) | Same | OK |
| Memory map at low address | Forces mmap buffer below 1GB (`boot.c:562-580`) | Uses EFI memory map in high memory | Adequate |

### Detailed Walkthrough

**PureOS ExitBootServices flow (`boot.c:549-655`):**

1. Calls `GetMemoryMap` to get size, then allocates the map buffer at a fixed low address (`boot.c:562-580`). It tries several candidate addresses (`0x3000000`, `0x2800000`, etc.) below 1GB so the kernel can read it through its identity map.
2. Calls `GetMemoryMap` a second time to fill the buffer (`boot.c:583`).
3. Stores the map address, entry count, and entry size in `boot_info_t` at `0x6000` (`boot.c:585-587`).
4. Calls `ExitBootServices(ImageHandle, mapKey)` (`boot.c:642`).
5. If it fails, retries up to 4 times with a fresh `GetMemoryMap` + `ExitBootServices` (`boot.c:646-654`).

**What Linux does differently:**
- Linux's `efi_stub` preserves UEFI runtime service pointers so the kernel can call `SetVirtualAddressMap()` later.
- Linux converts the EFI memory map into an E820-style map for the kernel's page table builder.
- Linux validates the memory map key more carefully and handles `EFI_CONVENTIONAL_MEMORY` vs `EFI_RESERVED_TYPE` vs `EFI_ACPI_RECLAIM_MEMORY` distinctly.

**Verdict: LOW RISK.** The handoff itself is reasonable. The gap is that UEFI runtime services are not preserved, but this does not cause early boot crashes. The memory map is passed to the kernel successfully.

---

## AREA 2: Early Memory / Paging

### Bootloader Page Tables (`src/boot/uefi/boot.c:78-125`)

PureOS uses **2MB huge pages** for the initial identity map:

```c
// boot.c:106-108 — Identity map first 1GB
for (int i = 0; i < 512; i++) {
    ((UINT64*)pd_low)[i] = (i * 0x200000) | 0x83; // Present, R/W, 2MB
}
```

**Gap 1: No cache attributes (PAT/PWT/PCD).**
The `0x83` flags mean Present + RW + PageSize. Linux sets `PWT=0, PCD=0, PAT=0` for normal memory but uses `PCD=1` or write-combining for MMIO. PureOS marks everything the same.

**Gap 2: LFB mapping overlap.**
The bootloader maps the framebuffer at `0xE0000000` (PDPT[3], PD index 256) using 2MB pages (`boot.c:116-122`). This occupies the same PDPT slot (index 3) as the `0xC0000000` higher-half kernel mapping. The code writes to `pd_high[256+i]` which maps `0xE0000000` → `0xC0000000 + 256*2MB` through `pd_high`. This is intentional (the 128MB block starting at `0xC0000000` covers both kernel and LFB), but it means the kernel's higher-half mapping covers `0xC0000000` through `0xE8000000` — a 640MB region all mapped via the same page directory.

### Kernel Page Tables (`src/kernel/hal/paging.c:72-249`)

**Gap 3: Identity map uses 4KB pages but no PAT/PCD control.**

```c
// paging.c:88-97
for (uint64_t i = 0; i < 0x40000000; i += 0x1000) {
    page_t *page = get_page(i, 1, kernel_pml4);
    page->present = 1;
    page->rw = 1;
    page->frame = i >> 12;
    // Missing: PWT, PCD, PAT, PGE flags
}
```

Every 4KB page in the 0-1GB identity map is mapped with default write-back caching. MMIO regions within this range (e.g., local APIC at 0xFEE00000, any PCI devices below 4GB) should be uncacheable but are mapped as cacheable. This can cause stale reads and deferred writes to device registers.

**Gap 4: kmalloc_ap physical address is 32-bit.**

```c
// heap.h:25
void *kmalloc_ap(size_t size, uint32_t *phys);
```

The heap allocator returns physical addresses as `uint32_t`, meaning all heap allocations are forced below 4GB. On QEMU this is fine. On the Lenovo LOQ with 16GB RAM, the UEFI memory map may place `EfiConventionalMemory` starting above 4GB, meaning `memmap_init()` at `memmap.c:38-66` will find **no usable memory in the 64MB-1GB range** and fall back to defaults.

The default `HEAP_START=0x4000000` (64MB) and `HEAP_SIZE=0x1C000000` (448MB) are hardcoded (`heap.h:10-11`), so the heap will exist — but only because the identity map covers 0-1GB. If UEFI firmware allocates critical structures in the 64MB-1GB range, the heap would overwrite them.

**Gap 5: memmap_init() memory map parsing.**

```c
// memmap.c:41-73
for (uint32_t i = 0; i < info->memory_map_entries; i++) {
    // ...
    if (info->boot_method == BOOT_METHOD_UEFI) {
        uint32_t *uefi_type = (uint32_t *)entry;
        if (*uefi_type == 7) { // EfiConventionalMemory
            usable = 1;
            uint64_t *phys = (uint64_t *)(map + i * info->memory_map_entry_size + 8);
            uint64_t *pages = (uint64_t *)(map + i * info->memory_map_entry_size + 24);
            entry->base_addr = *phys;
            entry->length = *pages * 4096;
        }
    }
```

The code assumes `EFI_MEMORY_DESCRIPTOR` layout: type at offset 0, physical address at offset 8, number of pages at offset 24. This matches the standard UEFI spec. However, it only looks for `EfiConventionalMemory` (type 7). It ignores `EfiBootServicesCode`, `EfiBootServicesData`, and `EfiReservedMemoryType` regions that might overlap with the identity-mapped range.

**Gap 6: No MMIO hole awareness.**
Linux parses the full UEFI memory map to identify reserved/MMIO regions (above 4GB PCI BARs, APIC, etc.) and skips them in page table construction. PureOS blindly identity-maps 0-1GB and then maps `0xF0000000-0xFFFFFFFF` for LAPIC/IOAPIC. If the Lenovo's MMIO layout differs (which it almost certainly does — modern Intel platforms have MMIO above 4GB), any MMIO region accessed outside these maps will page-fault.

**Verdict: HIGH RISK.** The 32-bit heap + blind 1GB identity map is the most fundamental architectural limitation. While it works in QEMU, real hardware has more complex memory layouts.

---

## AREA 3: PCI Enumeration

### PureOS (`src/drivers/pci.c:27-478`)

**Gap 7: Only uses legacy I/O port 0xCF8/0xCFC.**

```c
// pci.c:27-46
uint32_t address = (lbus << 16) | (lslot << 11) | (lfunc << 8) | (offset & 0xfc) | 0x80000000;
outl(PCI_CONFIG_ADDRESS, address);
tmp = inl(PCI_CONFIG_DATA);
```

Linux uses two methods:
1. **Legacy I/O ports (0xCF8/0xCFC):** Same as PureOS, limited to 256-byte config space.
2. **PCIe ECAM (Enhanced Configuration Access Mechanism):** Uses the MCFG ACPI table to get an MMIO base address, then memory-maps the entire config space. This allows access to config space up to 4096 bytes.

PureOS only uses legacy I/O. This means:
- Cannot access extended config space (registers >255).
- Cannot efficiently enumerate large numbers of devices.
- May miss devices that only respond to ECAM access.

**Gap 8: PCI bus scan limited to bus 0.**

```c
// pci.c:427
for (uint16_t bus = 0; bus < 1; bus++) { // BUS 0 ONLY!
```

On the Lenovo LOQ (Intel chipset), PCIe root ports are on bus 0, but devices behind PCIe root ports and switches appear on buses 1-255. Linux scans all 256 buses. The xHCI controller on the LOQ may be on a non-zero bus if it is behind a root port complex.

**If the xHCI is on bus 1+, PureOS will never find it.**

**Gap 9: No MCFG/ECAM support.**
Linux uses the ACPI MCFG table to get the MMIO base address for PCIe enhanced configuration access. PureOS never parses MCFG. On the Lenovo LOQ, if the xHCI controller is behind a PCIe port that only responds to ECAM, the legacy I/O port scan will miss it entirely.

**Gap 10: 64-bit BAR handling.**

```c
// pci.c:157-161
uint64_t orig_addr = (bar0 & 0xFFFFFFF0);
if (is_64bit) orig_addr |= ((uint64_t)bar1 << 32);
```

This reads the BAR correctly. But the mapping code at `pci.c:184-226` maps it to virtual `0x200000000` (8GB):

```c
// pci.c:192
mmio_base = 0x200000000ULL;
```

Then creates page table entries for virtual `0x200000000` pointing to the physical BAR address:

```c
// pci.c:196-215
for (uint64_t offset = 0; offset < 0x20000; offset += 0x1000) {
    uint64_t phys = (orig_addr & ~0xFFFULL) + offset;
    uint64_t virt = mmio_base + offset;
    struct { ... } *p = (void*)get_page(virt, 1, kernel_pml4);
    if (p) {
        p->present = 1; p->rw = 1; p->user = 0;
        p->nx = 1; p->pcd = 1; // Uncached
        p->frame = phys >> 12;
    }
}
```

On QEMU, this works because QEMU's virtual address space is flat. On real hardware, the xHCI MMIO BAR on the LOQ is likely at a **high physical address** (e.g., `0xFE000000` or above 4GB). The `get_page()` call should create the needed page table entries via `kmalloc_ap()`, which allocates from the heap below 1GB. This should work — but the PML4 entry for virtual `0x200000000` (PML4 index 4) needs to be created, and `get_page` with `make=1` should handle this.

**The real problem:** After the page tables are set up, CR3 must be reloaded to flush the TLB. PureOS does this at `pci.c:218-220`:

```c
uint64_t cr3;
__asm__ volatile("mov %%cr3, %0" : "=r" (cr3));
__asm__ volatile("mov %0, %%cr3" : : "r" (cr3) : "memory");
```

This flushes the TLB correctly. But the issue is that the **Rust `MemoryMapper`** (`xhci.rs:8-11`) assumes identity mapping:

```rust
unsafe fn map(&mut self, phys_base: usize, _bytes: usize) -> NonZeroUsize {
    NonZeroUsize::new(phys_base).unwrap() // Return physical as virtual
}
```

The `xhci::Registers::new(bar_ptr, mapper)` call in `xhci.rs:128` receives `bar_ptr` which is the virtual address `0x200000000`. The Rust `xhci` crate may internally call `mapper.map()` for its own register access calculations, receiving physical addresses and returning them as-is — but the CPU needs virtual addresses. This is a **potential virtual/physical confusion** inside the xHCI crate.

**Verdict: CRITICAL RISK.** The bus-0-only scan may miss the xHCI entirely. The MMIO mapping may work for addresses below 1GB but has virtual/physical confusion issues in the Rust layer.

---

## AREA 4: xHCI Ownership (THE CRASH POINT)

### PureOS xHCI Init Sequence

**Phase 1: C code discovery and handoff (`pci.c:152-401`)**

1. Detect xHCI controller: class 0x0C, subclass 0x03, progIF 0x30 (`pci.c:149-152`)
2. Read BAR0/BAR1 to get physical MMIO address (`pci.c:157-161`)
3. Map BAR to virtual `0x200000000` with PCD=1 (uncached) (`pci.c:184-226`)
4. Evaluate ACPI `_OSC` on `\_SB.PCI0` (`pci.c:233-276`)
5. Enable Bus Master + Memory Space, disable INTx (`pci.c:288-291`)
6. Walk extended capabilities for USB Legacy Support (`pci.c:322-397`)
7. BIOS-to-OS handoff: set OS Owned Semaphore (bit 24), poll until BIOS clears bit 16 (`pci.c:346-371`)
8. Clear USBLEGCTLSTS to disable all SMI sources (`pci.c:375-377`)
9. Call `rust_xhci_init(mmio_base)` (`pci.c:400-401`)

**Phase 2: Rust xHCI driver (`xhci.rs:118-780`)**

1. Create `xhci::Registers` from MMIO base (`xhci.rs:128`)
2. Read capability registers, validate caplength (`xhci.rs:130-135`)
3. **Second BIOS handoff** — "stealth takeover" (`xhci.rs:138-177`)
4. Stop controller: clear USBCMD.RunStop (`xhci.rs:187-203`)
5. Wait for USBSTS.HcHalted with 500ms timeout (`xhci.rs:193-203`)
6. Reset controller: set USBCMD.HCRST (`xhci.rs:209-224`)
7. Wait for reset complete with 1s timeout (`xhci.rs:216-224`)
8. Wait for CNR (Controller Not Ready) to clear with 1s timeout (`xhci.rs:225-235`)
9. Set Max Device Slots Enabled (`xhci.rs:242-250`)
10. Allocate DCBAAP, command ring, event ring (`xhci.rs:252-313`)
11. Configure Interrupter 0 (`xhci.rs:304-316`)
12. Start controller (`xhci.rs:336-355`)
13. Port scan and device enumeration (`xhci.rs:390-438`)

### Critical Gaps

**Gap 11: Double handoff — contradictory ownership protocol.**

The C code in `pci.c:346-384` does a proper handoff:
```c
// Set HC OS Owned Semaphore (bit 24)
*ext_cap = val | (1 << 24);
// Wait for BIOS to clear its ownership bit (bit 16)
while (timeout > 0) {
    val = *ext_cap;
    if (!((val >> 16) & 1)) break; // BIOS released!
    for (volatile int d = 0; d < 100000; d++) {}
    timeout--;
}
```

Then the Rust code in `xhci.rs:161-170` does a "stealth takeover":
```rust
// Force clear BIOS Owned Semaphore (Bit 16), but DO NOT set OS Owned Semaphore (Bit 24)
core::ptr::write_volatile(ptr as *mut u32, cap & !(1 << 16));
// Wait ~50ms
wait_ticks(13);
// Clear USBLEGCTLSTS
core::ptr::write_volatile(ctl_sts_ptr as *mut u32, 0);
```

This is **contradictory**: the C code already set the OS bit and waited for BIOS to release. The Rust code then clears the BIOS bit again and explicitly does NOT set the OS bit. On Intel platforms with SMM-based USB legacy emulation, the SMM handler monitors these bits. If the OS bit is cleared after being set, the SMM handler may interpret this as the OS relinquishing ownership, causing it to reassert BIOS control.

**Gap 12: No post-handoff delay.**

Linux's `xhci-pci.c` sequence:
1. Set OS Owned Semaphore
2. Poll until BIOS clears bit 16
3. Disable all SMI sources in USBLEGCTLSTS
4. **Wait 1 second** for SMM to settle
5. Then reset the controller

PureOS does steps 1-3 but **skips step 4 entirely**. The SMM handler may still be processing when PureOS resets the controller. This causes the CPU to enter SMM mid-reset, which can freeze the CPU or corrupt the OS state.

**Gap 13: xHCI reset during active SMI.**

If the BIOS handoff is incomplete (BIOS still owns the controller, or SMM is still processing the ownership change), writing to `USBCMD.HCRST` triggers an SMI. The SMM handler tries to service it, but the OS has already exited boot services and set up its own IDT/GDT. The SMM handler returns with corrupted OS state, causing a triple fault.

**Gap 14: MMIO read failure on real hardware.**

```rust
// xhci.rs:130
let caplength = registers.capability.caplength.read_volatile().get();
if caplength == 0 || caplength == 0xFF {
    // ABORT
}
```

If the xHCI BAR is at a physical address >1GB that is not properly mapped in the page tables, `read_volatile` will page-fault. The `MemoryMapper` returns the physical address as virtual, but the actual mapping is at virtual `0x200000000`. If the `xhci` crate internally translates offsets using the mapper, it will use the wrong virtual address.

**Gap 15: `rust_xhci_init` event ring uses virtual pointers for TRBs.**

```rust
// xhci.rs:1209
let base = XHCI_KB_INT_RING as *mut u32;
```

`XHCI_KB_INT_RING` is set from `kmalloc_ap`'s virtual return value. This is correct — TRBs should be accessed via virtual addresses. But the physical address (`XHCI_KB_INT_PHYS`) is used for the TRB pointer fields that the xHCI hardware reads. This is also correct. However, the event ring base (`XHCI_EVENT_RING_BASE`) is similarly set from `kmalloc_ap`'s virtual return:

```rust
// xhci.rs:321
XHCI_EVENT_RING_BASE = erst_seg_ptr as u64; // virtual
```

And the ERDP (Event Ring Dequeue Pointer) is set using the physical address:
```rust
// xhci.rs:1391
let new_erdp = XHCI_EVENT_RING_PHYS + (idx as u64) * 16;
```

But wait — `new_erdp` is calculated from `XHCI_EVENT_RING_PHYS` which is the **physical** address of the event ring segment (4096 bytes). The ERDP should point to a TRB within the event ring segment. Each TRB is 16 bytes, so `idx * 16` is correct. But `XHCI_EVENT_RING_PHYS` is the physical base of the 4096-byte segment, and `idx * 16` must not exceed 4096. With `ring_size=256` TRBs and 16 bytes each, the max is `255 * 16 = 4080`, which fits. This is correct.

However, the event ring segment table (ERST) entry 0 at `xhci.rs:299` sets the ring segment base address to the physical address of `erst_seg_ptr`:
```rust
erst.write_volatile(erst_seg_phys as u64);
```
And the ERDP initial value at `xhci.rs:312`:
```rust
interrupter.erdp.update_volatile(|w| {
    w.set_event_ring_dequeue_pointer(erst_seg_phys as u64);
});
```

This is correct — the hardware uses physical addresses for the ERST and ERDP.

**Verdict: CRITICAL RISK.** This is the most likely crash point. The double handoff, missing post-handoff delay, and SMM interaction on real hardware make this nearly guaranteed to fail.

---

## AREA 5: ACPI / SMM Interaction

### PureOS (`src/kernel/acpi.c:89-142`, `src/kernel/hal/acpi_osl.c:59-87`, `src/drivers/pci.c:233-276`)

**Gap 16: `_OSC` evaluation may change SMM behavior.**

```c
// pci.c:233-276
uint32_t caps[3] = {
    0x00000000, // Query Support (Bit 0) and reserved
    0x0000001F, // Support Field (Native Hot Plug, PME, AER, etc.)
    0x0000001F  // Control Field
};
```

On the Lenovo LOQ, `_OSC` is used by the firmware to negotiate PCIe/USB ownership. If PureOS sends the wrong capabilities or the firmware rejects the request, SMM behavior changes. The `_OSC` control field bits include:
- Bit 0: PCI Express Native Hot Plug control
- Bit 1: SHPC Native Hot Plug control
- Bit 2: PCIe PME
- Bit 3: PCIe Advanced Error Reporting
- Bit 4: PCIe ACS (Access Control Services)

PureOS requests control of ALL of these. Linux is much more conservative — it only requests capabilities it actually supports, and it checks the `_OSC` return buffer for denied capabilities. On the Lenovo LOQ, if the firmware denies some capabilities but PureOS assumes it has them, subsequent PCIe operations may fail.

**Gap 17: No `_OSI` evaluation.**

Linux evaluates `\_OSI` to inform the DSDT which OS is running:
```c
// Linux kernel: drivers/acpi/osl.c
acpi_osi_init();
// This calls AcpiEvaluateObject(ACPI_ROOT_OBJECT, "_OSI", &arg, NULL)
// with arg = "Windows 2020" (or similar)
```

Many laptops have DSDT conditional code that changes behavior based on `_OSI`. For example:
- USB power management may differ
- PCIe ASPM (Active State Power Management) settings change
- SMM behavior is conditioned on OS identity
- Fan curve and thermal management differ

Without `_OSI`, the DSDT uses a fallback path that may behave differently than expected. On the Lenovo LOQ, the DSDT likely has Windows-specific code paths that properly initialize USB legacy emulation. Without `_OSI`, these paths may not execute.

**Gap 18: `AcpiOsMapMemory` identity-mapping assumption.**

```c
// acpi_osl.c:59-87
void *AcpiOsMapMemory(ACPI_PHYSICAL_ADDRESS Where, ACPI_SIZE Length) {
    if (Where + Length <= 0x40000000ULL) {
        return (void *)(uintptr_t)Where; // Identity map < 1GB
    }
    if (Where >= 0xF0000000ULL && Where + Length <= 0x100000000ULL) {
        return (void *)(uintptr_t)Where; // MMIO region 0xF0000000-0xFFFFFFFF
    }
    // Dynamic identity mapping for addresses above 1GB
    // ...
}
```

If ACPICA needs to access tables or registers above 4GB (which modern Intel platforms frequently have), the dynamic mapping section at `acpi_osl.c:69-86` creates page table entries. This uses `get_page()` to allocate and map pages, which works — but the identity-mapped virtual addresses may conflict with the kernel's higher-half mapping if addresses fall in the `0xC0000000-0xFFFFFFFF` range.

**Gap 19: ACPICA table initialization order.**

```c
// acpi.c:95-123
AcpiInitializeSubsystem();
AcpiInitializeTables(NULL, 16, FALSE);
AcpiLoadTables();
AcpiEnableSubsystem(ACPI_FULL_INITIALIZATION);
AcpiInitializeObjects(ACPI_FULL_INITIALIZATION);
```

The `AcpiEnableSubsystem(ACPI_FULL_INITIALIZATION)` call transitions the platform from LEGACY to ACPI mode by writing `acpi_enable` to the SMI_CMD port. On the Lenovo LOQ, this triggers an SMI that initializes the ACPI-specific SMM handlers. If this happens before the xHCI handoff is complete, the SMM handler may conflict with the OS's xHCI ownership.

**Verdict: MEDIUM-HIGH RISK.** The `_OSC` evaluation and missing `_OSI` could cause firmware to leave SMM active or use unexpected code paths.

---

## AREA 6: Interrupt Controller Setup

### PureOS (`src/kernel/hal/pic.c:6-33`, `src/kernel/hal/apic.c:31-93`)

The init sequence is:

```c
// hal.c:33-51
void hal_init() {
    paging_init();      // Page tables
    acpi_init();        // ACPI tables (PIC still in firmware state)
    pic_init();         // Remap PIC, unmask IRQs
    lapic_init();       // Enable LAPIC, configure IOAPIC, then disable PIC
    smp_init();         // SMP init
}
```

**Gap 20: PIC is enabled during ACPI/PCI init.**

```c
// pic.c:27-30
outb(0x21, 0xF0); // Enable IRQ0,1,2,3,4,5,6,7 on master
outb(0xA1, 0xEB); // Enable IRQ12 (mouse) and IRQ10 (NE2000) on slave
```

Between `pic_init()` and `lapic_init()`, the PIC is fully active and unmasks several IRQs. During this window:
- Any stale PCI interrupt routed via the PIC will fire
- The ISR vectors are set (from `idt_init()` + `isr_install()`) but the IOAPIC is not yet configured
- On the Lenovo LOQ, the xHCI controller may have a legacy INTx interrupt routed to IRQ 10 (or similar) via the PIC
- If this fires between `pic_init()` and `lapic_init()`, the handler may not be ready

**Gap 21: No IOAPIC routing for xHCI.**

The IOAPIC routes GSI 1 (keyboard → vector 33), GSI 2 (PIT → vector 32), and GSI 12 (mouse → vector 44) (`apic.c:73-83`):

```c
rs_ioapic_route(2, 32, cpus[0].apic_id, false, false);  // PIT
rs_ioapic_route(1, 33, cpus[0].apic_id, false, false);  // Keyboard
rs_ioapic_route(12, 44, cpus[0].apic_id, false, false); // Mouse
```

But **does not route the xHCI IRQ**. The xHCI controller on the Lenovo LOQ is likely on GSI 16-23 (PCI MSI/MSI-X range). PureOS never routes these through the IOAPIC.

**Gap 22: INTx disabled but no MSI enabled.**

```c
// pci.c:290-291
cmd2 |= (1 << 10); // Bit 10 (Interrupt Disable) = 1: no pin IRQs
```

PureOS disables INTx but never enables MSI. Linux's `xhci-pci.c` explicitly sets up MSI via:
```c
// Linux: drivers/usb/host/xhci-pci.c
pci_alloc_irq_vectors(pdev, 1, 1, PCI_IRQ_MSI | PCI_IRQ_LEGACY);
```

On the Lenovo LOQ, the xHCI controller uses MSI by default. Without MSI:
- No xHCI interrupts are delivered to the CPU
- The event ring never advances via interrupt
- `rust_xhci_handle_irq` never fires from hardware
- The driver must rely entirely on timer-tick polling at 250 Hz
- During initialization, the controller waits for command completion events that never arrive
- Timeout paths fire, but the controller may be in an inconsistent state

**Gap 23: Stale interrupts from PIC mode.**

The PIC remapping at `pic.c:14-15` maps IRQ0 → vector 32, IRQ1 → vector 33, etc. If any hardware device generates a legacy interrupt before the IOAPIC takes over, the ISR vector may reference an unregistered handler. The `isr_install()` function registers ISR stubs for all 256 vectors, but the actual keyboard/mouse handlers are registered later in the driver init phase.

**Gap 24: No I/O APIC identity for LAPIC.**

The LAPIC is mapped at `local_apic_phys_addr` (from MADT, typically `0xFEE00000`). The kernel's paging maps `0xF0000000-0xFFFFFFFF` identity:

```c
// paging.c:153-160
for (uint64_t i = 0xF0000000; i < 0x100000000ULL; i += 0x1000) {
    page_t *page = get_page(i, 1, kernel_pml4);
    page->present = 1;
    page->rw = 1;
    page->frame = i >> 12;
}
```

This maps the LAPIC correctly for identity access. However, the `lapic_write` function at `apic.c:16-20` uses:
```c
volatile uint32_t *apic = (volatile uint32_t *)(uintptr_t)(local_apic_phys_addr + reg);
*apic = data;
```

This relies on the identity map. If `local_apic_phys_addr` is not in the `0xF0000000-0xFFFFFFFF` range (some platforms put it elsewhere), this will page-fault.

**Verdict: HIGH RISK.** No MSI setup means xHCI interrupts don't work. This causes the port scan to rely entirely on polling, which partially works but may miss critical events during initialization.

---

## AREA 7: What Linux Does That PureOS Doesn't (Summary Table)

| Subsystem | Linux | PureOS | Impact |
|-----------|-------|--------|--------|
| **Memory map** | Parses full UEFI memory map, skips reserved/MMIO regions, builds page tables with correct cache attributes (UC for MMIO, WB for RAM) | Blind 1GB identity map + hardcoded `0xF0000000` MMIO region | MMIO access to unmapped regions causes page fault |
| **PCI scan** | All 256 buses, MCFG/ECAM support, extended config space access | Bus 0 only, legacy I/O ports (0xCF8/0xCFC) | May miss devices on non-zero buses |
| **xHCI handoff** | One-shot: set OS bit, poll BIOS clear, disable SMI, **1-second delay**, then reset | Double handoff (C + Rust), no delay, potential SMM re-entry | SMM traps the reset, CPU freezes |
| **MSI/MSI-X** | `pci_alloc_irq_vectors()` with MSI/MSI-X/legacy fallback | Never enables MSI, disables INTx | xHCI interrupts never fire from hardware |
| **ACPI _OSI** | Evaluates `_OSI("Windows 2020")` to get correct DSDT behavior | Never evaluates `_OSI` | DSDT uses wrong code path, USB/SMM behavior differs |
| **xHCI init** | Reset, wait CNR, configure MMIO, rings, port scan — all with proper cache attributes | MMIO mapped via virtual 0x200000000, Rust mapper returns physical as virtual | Potential virtual/physical confusion in xHCI crate |
| **Cache control** | PAT-based UC/WC/WB for MMIO vs RAM | All memory mapped with default WB (except xHCI BAR which has PCD=1) | MMIO writes may be cached and never reach hardware |
| **Post-handoff delay** | 1 second SMM settle time after USB Legacy handoff | Immediate reset | SMM interrupt during reset causes triple fault |
| **`_OSC` negotiation** | Conservative, requests only needed capabilities, checks denied bits | Requests all capabilities, ignores denial | Firmware may refuse or change SMM behavior |

---

## RANKED CRASH CAUSES (Most Likely → Least Likely)

### 1. xHCI BIOS Handoff + SMM Interaction [CRITICAL]

**Files:** `src/drivers/pci.c:322-397`, `rust/src/xhci.rs:138-176`

**Root cause:** The double handoff creates an inconsistent ownership state. The C code sets OS Owned Semaphore (bit 24) and waits for BIOS to clear bit 16. Then the Rust code clears bit 16 again WITHOUT setting bit 24. On the Lenovo LOQ's Intel chipset, the SMM handler monitors USBLEGSUP. When PureOS resets the xHCI controller without completing the proper handoff sequence, the SMM handler traps the reset command, enters SMM mode, and the CPU either freezes in SMM or returns to corrupted OS state.

Additionally, the missing 1-second post-handoff delay means SMM is still processing when the controller reset is issued.

**Fix:** Remove the Rust-side stealth handoff. Do a single, proper handoff in C with a 1-second post-handoff delay, then pass the MMIO base to Rust. Match Linux's `xhci-pci.c:usb_do_external_hub()` sequence exactly.

### 2. Missing MSI/MSI-X Setup [HIGH]

**Files:** `src/drivers/pci.c:288-291`

**Root cause:** Disabling INTx without enabling MSI means the xHCI controller has no way to deliver interrupts via hardware. The timer-tick polling at 250 Hz partially compensates but misses events during the critical initialization window. Command completion events, port status changes, and transfer events may all be missed.

**Fix:** Implement `pci_alloc_irq_vectors()` equivalent that queries MSI capability from config space and programs the IOAPIC/APIC for MSI delivery.

### 3. PCI Bus Scan Too Narrow [HIGH]

**File:** `src/drivers/pci.c:427`

**Root cause:** Scanning only bus 0 will miss xHCI if it is behind a PCIe root port on the LOQ. The LOQ's Intel chipset routes USB through a root port complex that may place the xHCI controller on bus 1 or higher.

**Fix:** Scan all 256 buses. Implement proper PCI topology enumeration (check multi-function devices, PCI-to-PCI bridges).

### 4. MMIO Mapping Virtual/Physical Confusion [MEDIUM-HIGH]

**Files:** `src/drivers/pci.c:184-226`, `rust/src/xhci.rs:8-11`

**Root cause:** The C code maps the xHCI BAR to virtual `0x200000000`, but the Rust `MemoryMapper` returns physical addresses as virtual. The `xhci::Registers::new()` receives the virtual base (`0x200000000`) and adds register offsets to it — this part works. But the Rust `xhci` crate's internal `Mapper::map()` calls (if any exist) would get wrong addresses, leading to MMIO access at incorrect virtual addresses.

**Fix:** Either identity-map the xHCI BAR (simpler) or ensure the Rust mapper returns `0x200000000` as the base for all internal translations.

### 5. Missing `_OSI` Evaluation [MEDIUM]

**File:** `src/kernel/acpi.c` (not present)

**Root cause:** The DSDT on the Lenovo LOQ has conditional code paths for different operating systems. Without `_OSI`, the firmware may leave USB in a state that conflicts with xHCI ownership, or may use a DSDT code path that does not properly initialize USB legacy emulation.

**Fix:** Add `AcpiEvaluateObject(ACPI_ROOT_OBJECT, "_OSI", ...)` with "Windows 2020" (or "Linux") before any other ACPI evaluation.

### 6. PIC Enabled During Critical Window [MEDIUM]

**Files:** `src/kernel/hal/hal.c:42-46`, `src/kernel/hal/pic.c:27-30`

**Root cause:** Between `pic_init()` and `lapic_init()`, the PIC is fully active and unmasks several IRQs. A stale interrupt during this window (especially IRQ 10 which PureOS unmaps on the slave PIC) could vector to an unset handler.

**Fix:** Move `pic_init()` after `lapic_init()`, or mask all PIC interrupts immediately before enabling APIC.

### 7. No Cache Attribute Control for MMIO [LOW-MEDIUM]

**Files:** `src/kernel/hal/paging.c:88-97`, `src/drivers/pci.c:196-215`

**Root cause:** xHCI MMIO registers should be mapped with uncacheable (PCD=1) or write-combining (PAT) attributes. PureOS maps the xHCI BAR correctly (PCD=1 at `pci.c:211`), but the kernel's `paging_init()` at `paging.c:88-97` maps all 0-1GB identity memory with default write-back caching. Any other device memory in this range will have incorrect cache attributes.

**Fix:** Set PCD=1 and PAT=0 (UC) for all MMIO mappings in the identity map. Use the UEFI memory map to distinguish MMIO regions from RAM.

### 8. `_OSC` Capability Negotiation [LOW-MEDIUM]

**Files:** `src/drivers/pci.c:233-276`

**Root cause:** PureOS requests all PCIe capabilities (`0x0000001F`) without checking which ones the firmware actually grants. Linux checks the returned capability bits and adjusts its behavior for denied capabilities. If the Lenovo LOQ's firmware denies certain capabilities but PureOS assumes it has them, subsequent PCIe operations (hot plug, AER, etc.) may fail or trigger unexpected firmware behavior.

**Fix:** Check the `_OSC` return buffer for denied capabilities and adjust behavior accordingly.

---

## RECOMMENDED FIX PRIORITY

| Priority | Fix | Risk Addressed | Effort |
|----------|-----|----------------|--------|
| **P0** | Fix xHCI handoff: single handoff in C, 1s delay, no Rust re-handoff | Crash cause #1 | Low |
| **P1** | Enable MSI: query MSI capability, program IOAPIC/APIC | Crash cause #2 | Medium |
| **P1** | Scan all PCI buses (change `bus < 1` to `bus < 256`) | Crash cause #3 | Trivial |
| **P2** | Add `_OSI` evaluation | Crash cause #5 | Trivial |
| **P2** | Fix MMIO cache attributes (PCD/PAT) | Crash cause #7 | Low |
| **P2** | Fix `_OSC` negotiation (check return buffer) | Crash cause #8 | Low |
| **P3** | Fix PIC enabled during critical window | Crash cause #6 | Trivial |
| **P3** | Fix MMIO virtual/physical confusion in Rust layer | Crash cause #4 | Medium |

---

## APPENDIX: File Reference

| File | Lines | Purpose |
|------|-------|---------|
| `src/boot/uefi/boot.c` | 1-683 | UEFI bootloader: page table setup, ExitBootServices, GDT, jump to kernel |
| `src/kernel/kernel.c` | 1-1225 | Kernel main init sequence, driver init order |
| `src/kernel/hal/paging.c` | 1-436 | Kernel page table setup, identity map, higher-half, page fault handler |
| `src/drivers/pci.c` | 1-478 | PCI enumeration (legacy I/O), xHCI discovery, BIOS handoff, _OSC |
| `src/kernel/acpi.c` | 1-193 | ACPICA initialization, MADT parsing, shutdown/reboot |
| `src/kernel/hal/acpi_osl.c` | 1-262 | ACPI OS layer: memory mapping, PCI config access, spinlocks |
| `rust/src/xhci.rs` | 1-1475 | Rust xHCI driver: controller reset, port scan, HID keyboard |
| `src/kernel/hal/hal.c` | 1-58 | HAL init: GDT, IDT, paging, ACPI, PIC, APIC, SMP |
| `src/kernel/hal/apic.c` | 1-104 | Local APIC + IOAPIC setup, IRQ routing |
| `src/kernel/hal/pic.c` | 1-33 | Legacy PIC remapping and IRQ masking |
| `src/kernel/hal/idt.c` | 1-29 | IDT setup |
| `src/kernel/hal/gdt.c` | 1-129 | GDT + TSS setup |
| `src/kernel/memmap.c` | 1-90 | UEFI memory map parsing for heap placement |
| `src/kernel/heap.h` | 1-40 | Heap configuration, kmalloc_ap (32-bit phys) |
| `src/boot/boot_info.h` | 1-32 | Boot info structure passed from bootloader to kernel |
| `build.bat` | 1-523+ | Compiler flags, `-DPUREOS_OS_OWNS_XHCI` enabled |
| `run_os.bat` | 1-5 | QEMU flags: 4GB RAM, OVMF UEFI, bochs-display, qemu-xhci |

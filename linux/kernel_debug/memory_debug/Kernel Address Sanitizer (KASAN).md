# Kernel Address Sanitizer (KASAN)

## 总结

KASAN 用来检测**越界**和**释放后使用**，

当前配置

    CONFIG_CMDLINE="kasan=on kasan.fault=panic kasan.stacktrace=on kasan.vmalloc=on"
    CONFIG_KASAN=y
    CONFIG_KASAN_EXTRA_INFO=y
    CONFIG_KASAN_GENERIC=y
    CONFIG_KASAN_OUTLINE=y
    CONFIG_KASAN_STACK=y
    CONFIG_KASAN_VMALLOC=y
    CONFIG_KASAN_MODULE_TEST=m
    CONFIG_KASAN_EXTRA_INFO=y

## 概述

内核地址清理器（KASAN）是一种动态内存安全错误检测器，用来检测**越界**`out-of-bounds`和**释放后使用**`use-after-free`的错误

##### KASAN 三种模式

1.  **通用 KASAN** (GCC 8.3.0+)

    通过 CONFIG\_KASAN\_GENERIC 启用，类似于用户空间检测 ASan，性能和内存开销较大。

    **检测内存类型**：slab、page\_alloc、vmap、vmalloc、栈和全局内存

2.  **软件 Tag KASAN** (GCC 11+)

    通过 CONFIG\_KASAN\_SW\_TAGS 启用，类似用户空间 HWASan，仅支持 arm64,内存开销适中

    **检测内存类型**：slab、page\_alloc、vmalloc、栈内存

3.  **硬件 Tag KASAN** (GCC 10+ or Clang 12+)

    通过 CONFIG\_KASAN\_HW\_TAGS 启用，用作现场内存漏洞检测器或者安全缓解措施的模式。此模式仅适用于支持 [MTE](https://docs.kernel.org/arch/arm64/memory-tagging-extension.html)(Memory Tagging Extension)的 arm64 CPU，性能和内存开销低。

    **检测内存类型**：slab、page\_alloc 和非可执行 vmalloc 内存

    > \[!NOTE]
    >
    > 补充对比图表

##### 启用CONFIG方法

基础配置 `CONFIG_KASAN=y` ， `CONFIG_KASAN_GENERIC` 启用通用 KASAN， `CONFIG_KASAN_SW_TAGS` 启用软件 Tag KASAN,  `CONFIG_KASAN_HW_TAGS` 启用硬件 Tag KASAN。

对于 软件 Tag KASAN，还需要选择 `CONFIG_KASAN_OUTLINE` 或 `CONFIG_KASAN_INLINE` ，Outline 和 inline 是编译器插桩类型。前者生成的二进制文件更小，而后者速度最高可达2倍。

`CONFIG_STACKTRACE` ：跟踪 slab 分配和释放的堆栈

`CONFIG_PAGE_OWNER` ：要包跟踪 pages 分配和释放的堆栈，启动时需要传入 `page_owner=on`

`CONFIG_KASAN_EXTRA_INFO` ：报告更多信息，目前包括分配和释放的CPU和时间戳

##### 启动参数

\- `panic_on_warn=1` ：会在打印错误报告后触发内核 panic。

\- `kasan_multi_shot` ：默认情况下 KASAN 只会在第一次异常时打印错误报告，使用这个参数时，每次都会打印错误报告。并且这个会禁用 `panic_on_warn` 功能

\- `kasan.fault=report/panic/panic_on_write` ：控制是仅打印报告、触发内核 panic，还是仅在无效写入时 panic。这个会覆盖 `kasan_multi_shot`

软件或者硬件 Tag Kasan 支持修改堆栈跟踪行为

\- `kasan.stacktrace=off/on` ：禁用或启用分配和释放堆栈跟踪收集（默认：`on`）。

\- `kasan.stack_ring_size=<条目数量>` ：指定堆栈环中的条目数量（默认：`32768`）。

硬件 Tag Kasan 模式支持启动参数

\- `kasan=off/on` ：控制是否启用 KASAN

\- `kasan.mode=sync/async/asymm` ：控制KASAN 是否配置为同步、异步还是不对称执行模式

   同步模式：当标签检查故障发生时立即检测到错误访问。

   异步模式：错误访问检测被延迟。当标签检查故障发生时，信息存储在硬件中（对于 arm64，存储在 TFSR\_EL1 寄存器中）。内核定期检查硬件，只在这些检查期间报告标签故障。

   不对称模式：读取时同步检测错误访问，写入时异步检测错误访问。

\- `kasan.write_only=off` 或 `kasan.write_only=on` 控制 KASAN 是仅检查写入（存储）访问还是所有访问（默认：`off`）。

\- `kasan.vmalloc=off` 或 `=on` 禁用或启用 vmalloc 分配的标签（默认：`on`）。

\- `kasan.page_alloc.sample=<采样间隔>` 使 KASAN 只对第 N 个 page\_alloc 分配加上标签，其中分配的大小等于或大于  `kasan.page_alloc.sample.order`，N 是 `sample` 参数的值（默认：`1`，即为每个此类分配加标签）。  此参数旨在减轻 KASAN 引入的性能开销。

\- `kasan.page_alloc.sample.order=<最小页阶数>` 指定受采样影响的最小分配阶数（默认：`3`）。仅当 `kasan.page_alloc.sample` 设置为大于 `1` 的值时适用。

##### 错误报告

```tex
[160210.815534][    C0] ==================================================================
[160210.815552][    C0] BUG: KASAN: slab-out-of-bounds in dump_tracking+0xa4/0x41c [minidump]
[160210.815705][    C0] Read of size 4 at addr ffffff886d65c098 by task android.bg/5006
[160210.815746][    C0] CPU: 0 UID: 1000 PID: 5006 Comm: android.bg Tainted: G        W  OE      6.12.23-android16-5-g2edd335635f3-mi-4k #1 16ca076596b1c2d5b43714d72fa3a7295ca9d1f8
[160210.815776][    C0] Tainted: [W]=WARN, [O]=OOT_MODULE, [E]=UNSIGNED_MODULE
[160210.815784][    C0] Hardware name: Qualcomm Technologies, Inc. Popsicle based on SM8850 (DT)
[160210.815795][    C0] Call trace:
[160210.815801][    C0]  dump_backtrace+0x120/0x170
[160210.815819][    C0]  show_stack+0x2c/0x40
[160210.815833][    C0]  dump_stack_lvl+0x84/0xb4
[160210.815848][    C0]  print_report+0x144/0x7a4
[160210.815872][    C0]  kasan_report+0xe0/0x140
[160210.815895][    C0]  __asan_load4+0x9c/0xa4
[160210.815907][    C0]  dump_tracking+0xa4/0x41c [minidump 6b45965016fdc7e3149f35d5e4f16917756dc70e]
[160210.816048][    C0]  get_each_kmemcache_object+0xcc/0x204
[160210.816067][    C0]  md_dump_memory+0x15fc/0x1d44 [minidump 6b45965016fdc7e3149f35d5e4f16917756dc70e]
[160210.816208][    C0]  md_dump_process+0xce4/0xd84 [minidump 6b45965016fdc7e3149f35d5e4f16917756dc70e]
[160210.816348][    C0]  qcom_wdt_bark_handler+0x218/0x2b8 [qcom_wdt_core 76fc84980aefa5e640f1a1231c8bc3073e898680]
[160210.816397][    C0]  __handle_irq_event_percpu+0xa8/0x2e4
[160210.816414][    C0]  handle_irq_event+0x5c/0xe0
[160210.816430][    C0]  handle_fasteoi_irq+0x278/0x528
[160210.816451][    C0]  generic_handle_domain_irq+0xac/0xd4
[160210.816466][    C0]  gic_handle_irq+0x5c/0x148
[160210.816479][    C0]  call_on_irq_stack+0x3c/0x50
[160210.816497][    C0]  do_interrupt_handler+0x74/0xc0
[160210.816518][    C0]  el1_interrupt+0x34/0x58
[160210.816533][    C0]  el1h_64_irq_handler+0x18/0x24
[160210.816546][    C0]  el1h_64_irq+0x68/0x6c
[160210.816559][    C0]  queued_spin_lock_slowpath+0x518/0x680
[160210.816574][    C0]  _raw_spin_lock+0x134/0x13c
[160210.816586][    C0]  binder_ioctl_write_read+0x1020/0x392c
[160210.816607][    C0]  binder_ioctl+0x304/0x1194
[160210.816629][    C0]  __arm64_sys_ioctl+0xec/0x14c
[160210.816645][    C0]  invoke_syscall+0x90/0x1e8
[160210.816658][    C0]  el0_svc_common+0x134/0x168
[160210.816671][    C0]  do_el0_svc+0x34/0x44
[160210.816683][    C0]  el0_svc+0x38/0x84
[160210.816696][    C0]  el0t_64_sync_handler+0x70/0xbc
[160210.816710][    C0]  el0t_64_sync+0x19c/0x1a0
[160210.817017][    C0] Allocated by task 3202 on cpu 6 at 160210.753550s:
[160210.817040][    C0]  kasan_save_track+0x44/0x9c
[160210.817072][    C0]  kasan_save_alloc_info+0x40/0x54
[160210.817097][    C0]  __kasan_kmalloc+0x9c/0xb8
[160210.817129][    C0]  __kmalloc_cache_noprof+0x1fc/0x444
[160210.817158][    C0]  kernfs_fop_open+0x3fc/0x6f0
[160210.817187][    C0]  do_dentry_open+0x50c/0x9c4
[160210.817214][    C0]  vfs_open+0x98/0x1d8
[160210.817239][    C0]  path_openat+0x12f0/0x16c8
[160210.817269][    C0]  do_filp_open+0x134/0x258
[160210.817299][    C0]  do_sys_openat2+0xfc/0x18c
[160210.817324][    C0]  __arm64_sys_openat+0xc4/0xf8
[160210.817350][    C0]  invoke_syscall+0x90/0x1e8
[160210.817372][    C0]  el0_svc_common+0xb8/0x168
[160210.817395][    C0]  do_el0_svc+0x34/0x44
[160210.817417][    C0]  el0_svc+0x38/0x84
[160210.817439][    C0]  el0t_64_sync_handler+0x70/0xbc
[160210.817462][    C0]  el0t_64_sync+0x19c/0x1a0
[160210.817499][    C0] Last potentially related work creation:
[160210.817514][    C0]  kasan_save_stack+0x40/0x70
[160210.817544][    C0]  __kasan_record_aux_stack+0xb0/0xcc
[160210.817569][    C0]  kasan_record_aux_stack_noalloc+0x14/0x24
[160210.817594][    C0]  kvfree_call_rcu+0x30/0x4f0
[160210.817624][    C0]  kernfs_unlink_open_file+0x1e8/0x228
[160210.817652][    C0]  kernfs_fop_release+0x148/0x18c
[160210.817679][    C0]  __fput+0xf8/0x438
[160210.817708][    C0]  __fput_sync+0x74/0x84
[160210.817738][    C0]  __arm64_sys_close+0xe4/0x140
[160210.817764][    C0]  invoke_syscall+0x90/0x1e8
[160210.817787][    C0]  el0_svc_common+0xb8/0x168
[160210.817808][    C0]  do_el0_svc+0x34/0x44
[160210.817830][    C0]  el0_svc+0x38/0x84
[160210.817851][    C0]  el0t_64_sync_handler+0x70/0xbc
[160210.817875][    C0]  el0t_64_sync+0x19c/0x1a0
[160210.817911][    C0] The buggy address belongs to the object at ffffff886d65c020
[160210.817911][    C0]  which belongs to the cache kmalloc-96 of size 96
[160210.817934][    C0] The buggy address is located 48 bytes to the right of
[160210.817934][    C0]  allocated 72-byte region [ffffff886d65c020, ffffff886d65c068)
[160210.817974][    C0] The buggy address belongs to the physical page:
[160210.817991][    C0] page: refcount:1 mapcount:0 mapping:0000000000000000 index:0xffffff886d65d120 pfn:0x8ed65c
[160210.818016][    C0] head: order:1 mapcount:0 entire_mapcount:0 nr_pages_mapped:0 pincount:0
[160210.818038][    C0] flags: 0x4000000000000240(workingset|head|zone=1)
[160210.818061][    C0] page_type: f5(slab)
[160210.818085][    C0] raw: 4000000000000240 ffffff883770c3c0 fffffffec1340510 ffffff8837700250
[160210.818112][    C0] raw: ffffff886d65d120 0000000000200018 00000001f5000000 0000000000000000
[160210.818138][    C0] head: 4000000000000240 ffffff883770c3c0 fffffffec1340510 ffffff8837700250
[160210.818164][    C0] head: ffffff886d65d120 0000000000200018 00000001f5000000 0000000000000000
[160210.818189][    C0] head: 4000000000000001 fffffffee1b59701 ffffffffffffffff 0000000000000000
[160210.818215][    C0] head: 0000000000000002 0000000000000000 00000000ffffffff 0000000000000000
[160210.818234][    C0] page dumped because: kasan: bad access detected
[160210.818251][    C0] page_owner tracks the page as allocated
[160210.818266][    C0] page last allocated via order 1, migratetype Unmovable, gfp_mask 0x1d2040(__GFP_IO|__GFP_NOWARN|__GFP_NORETRY|__GFP_COMP|__GFP_NOMEMALLOC|__GFP_HARDWALL), pid 15272, tgid 15272 (oid.photopicker), ts 133749823220, free_ts 133700713688
[160210.818305][    C0]  post_alloc_hook+0x240/0x264
[160210.818335][    C0]  prep_new_page+0x2c/0xf8
[160210.818363][    C0]  get_page_from_freelist+0x1ca4/0x1d2c
[160210.818395][    C0]  __alloc_pages_noprof+0x2d8/0x6c0
[160210.818425][    C0]  alloc_slab_page+0x84/0x21c
[160210.818455][    C0]  allocate_slab+0x7c/0x390
[160210.818485][    C0]  ___slab_alloc+0x788/0xbb8
[160210.818514][    C0]  __kmalloc_noprof+0x31c/0x584
[160210.818542][    C0]  fscrypt_fname_encrypt+0x194/0x390
[160210.818571][    C0]  fscrypt_setup_filename+0x1ec/0x4b0
[160210.818599][    C0]  __fscrypt_prepare_lookup+0x38/0x12c
[160210.818628][    C0]  f2fs_prepare_lookup+0x104/0x1d8
[160210.818655][    C0]  f2fs_lookup+0xf0/0x7d8
[160210.818684][    C0]  __lookup_slow+0x1c4/0x2a0
[160210.818714][    C0]  lookup_slow+0x48/0x70
[160210.818743][    C0]  link_path_walk+0x4c0/0x64c
[160210.818775][    C0] page last free pid 10284 tgid 8508 stack trace:
[160210.818793][    C0]  free_unref_page+0x7d4/0x928
[160210.818822][    C0]  __free_pages+0xcc/0x2a0
[160210.818852][    C0]  __free_slab+0xb0/0xcc
[160210.818880][    C0]  free_slab+0x24/0xec
[160210.818909][    C0]  free_to_partial_list+0x504/0x7a8
[160210.818940][    C0]  __slab_free+0x2cc/0x314
[160210.818970][    C0]  ___cache_free+0x14c/0x17c
[160210.818998][    C0]  qlink_free+0x44/0x90
[160210.819022][    C0]  qlist_free_all+0x40/0xa4
[160210.819047][    C0]  kasan_quarantine_reduce+0x120/0x144
[160210.819073][    C0]  __kasan_slab_alloc+0x2c/0x8c
[160210.819105][    C0]  kmem_cache_alloc_noprof+0x1a0/0x428
[160210.819133][    C0]  f2fs_init_casefolded_name+0xbc/0x200
[160210.819159][    C0]  __f2fs_setup_filename+0xb8/0x158
[160210.819184][    C0]  f2fs_prepare_lookup+0x188/0x1d8
[160210.819209][    C0]  f2fs_lookup+0xf0/0x7d8
[160210.819252][    C0] Memory state around the buggy address:
[160210.819270][    C0]  ffffff886d65bf80: fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc
[160210.819291][    C0]  ffffff886d65c000: fc fc fc fc 00 00 00 00 00 00 00 00 00 fc fc fc
[160210.819313][    C0] >ffffff886d65c080: fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc
[160210.819332][    C0]                             ^
[160210.819350][    C0]  ffffff886d65c100: fc fc fc fc fa fb fb fb fb fb fb fb fb fb fb fb
[160210.819371][    C0]  ffffff886d65c180: fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc
[160210.819390][    C0] ==================================================================
```

##### 实现细节

###### Generic KASAN

&#x9;   KASAN的原理是利用额外的内存标记可用内存的状态。这部分额外的内存被称作shadow memory（影子区）。KASAN将1/8的内存用作shadow memory。使用特殊的magic num填充shadow memory，在每一次load/store（load/store检查指令由编译器插入）内存的时候检测对应的shadow memory确定操作是否valid。连续8 bytes内存（8 bytes align）使用1 byte shadow memory标记。如果8 bytes内存都可以访问，则shadow memory的值为0；如果连续N(1 =< N <= 7) bytes可以访问，则shadow memory的值为N；如果8 bytes内存访问都是invalid，则shadow memory的值为负数。

<img src="https://github.com/ysg1011/ysg1011.github.io/blob/main/images/KASAN_shadow_memory.png?raw=true" style="zoom:100%;" />

###### Software Tag-Based KASAN

软件 KASAN（Software Tag-Based KASAN）使用软件内存标签的方法来检查访问有效性。它目前仅在 arm64 架构上实现。

软件 KASAN 利用 arm64 CPU 的顶部字节忽略（Top Byte Ignore, TBI）特性，在内核指针的顶部字节中存储指针标签。它使用影子内存来存储与每个 16字节内存单元相关联的内存标签（因此，它将内核内存的 1/16 专用于影子内存）。

在每次内存分配时，软件 KASAN 生成一个随机标签，用该标签标记已分配的内存，并将相同的标签嵌入到返回的指针中。

软件 KASAN 使用编译时检测在每次内存访问前插入检查。这些检查确保被访问内存的标签与用于访问该内存的指针标签相等。如果标签不匹配，软件标签型 KASAN 将打印错误报告。

软件 KASAN 使用 0xFF 作为匹配所有指针的标签（通过带有 0xFF 指针标签的指针进行的访问不会被检查）。值 0xFE 当前被保留用于标记已释放的内存区域。

###### Hardware Tag-Based KASAN

硬件 KASAN（Hardware Tag-Based KASAN）在概念上与软件模式相似，但使用硬件内存标签支持而不是编译器检测和影子内存。

硬件 KASAN 目前仅在 arm64 架构上实现，基于在 ARMv8.5 指令集架构中引入的 arm64 内存标签扩展（Memory Tagging Extension, MTE）和顶部字节忽略（TBI）。

使用特殊的 arm64 指令为每个分配分配内存标签。相同的标签被分配给指向这些分配的指针。在每次内存访问时，硬件确保被访问内存的标签与用于访问该内存的指针标签相等。如果标签不匹配，会生成故障并打印报告。

---
title: Bolt中Relocation作用分析
date: 2022-06-24 19:25:35
categories: compiler
tags: [bolt,post link optimizer,pgo]
---
## Relocation Section的必要性分析
Bolt插桩/优化过程中，要求其输入的二进制文件需包含relocation section。而正常编译出的二进制文件，linker 默认不输出这些section。下面就以coremark为例，探索Relocation Section在Bolt中的作用。

以coremark为例，默认编译出的二进制文件是不包含.rela.text section的。编译命令：
```shell
~/project/llvm/build/bin/clang -O2 -Ilinux -Iposix -I. -DFLAGS_STR=\""-O2 -DPERFORMANCE_RUN=1  -lrt"\" -DITERATIONS=0 -DPERFORMANCE_RUN=1 core_list_join.c core_main.c core_matrix.c core_state.c core_util.c posix/core_portme.c -o ./coremark.exe -lrt
```
查看其section `readelf -S coremark.exe` 如下：
```shell
There are 39 section headers, starting at offset 0xa170:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .interp           PROGBITS         00000000000002a8  000002a8
       000000000000001c  0000000000000000   A       0     0     1
  [ 2] .note.ABI-tag     NOTE             00000000000002c4  000002c4
       0000000000000020  0000000000000000   A       0     0     4
  [ 3] .gnu.hash         GNU_HASH         00000000000002e8  000002e8
       0000000000000024  0000000000000000   A       4     0     8
  [ 4] .dynsym           DYNSYM           0000000000000310  00000310
       0000000000000138  0000000000000018   A       5     1     8
  [ 5] .dynstr           STRTAB           0000000000000448  00000448
       00000000000000c8  0000000000000000   A       0     0     1
  [ 6] .gnu.version      VERSYM           0000000000000510  00000510
       000000000000001a  0000000000000002   A       4     0     2
  [ 7] .gnu.version_r    VERNEED          0000000000000530  00000530
       0000000000000030  0000000000000000   A       5     1     8
  [ 8] .rela.dyn         RELA             0000000000000560  00000560
       0000000000000288  0000000000000018   A       4     0     8
  [ 9] .rela.plt         RELA             00000000000007e8  000007e8
       00000000000000a8  0000000000000018  AI       4    30     8
  [10] .init             PROGBITS         0000000000001000  00001000
       000000000000001b  0000000000000000  AX       0     0     4
  [11] .rela.init        RELA             0000000000000000  00008740
       0000000000000018  0000000000000018   I      36    10     8
  [12] .plt              PROGBITS         0000000000001020  00001020
       0000000000000080  0000000000000010  AX       0     0     16
  [13] .plt.got          PROGBITS         00000000000010a0  000010a0
       0000000000000008  0000000000000008  AX       0     0     8
  [14] .text             PROGBITS         00000000000010b0  000010b0
       0000000000003af5  0000000000000000  AX       0     0     16
  [15] .rela.text        RELA             0000000000000000  00008758
       0000000000001128  0000000000000018   I      36    14     8
  [16] .fini             PROGBITS         0000000000004ba8  00004ba8
       000000000000000d  0000000000000000  AX       0     0     4
  [17] .rodata           PROGBITS         0000000000005000  00005000
       00000000000005d0  0000000000000000   A       0     0     8
  [18] .rela.rodata      RELA             0000000000000000  00009880
       0000000000000180  0000000000000018   I      36    17     8
  [19] .eh_frame_hdr     PROGBITS         00000000000055d0  000055d0
       000000000000017c  0000000000000000   A       0     0     4
  [20] .eh_frame         PROGBITS         0000000000005750  00005750
       0000000000000750  0000000000000000   A       0     0     8
  [21] .rela.eh_frame    RELA             0000000000000000  00009a00
       0000000000000420  0000000000000018   I      36    20     8
  [22] .init_array       INIT_ARRAY       0000000000007d50  00006d50
       0000000000000008  0000000000000008  WA       0     0     8
  [23] .rela.init_array  RELA             0000000000000000  00009e20
       0000000000000018  0000000000000018   I      36    22     8
  [24] .fini_array       FINI_ARRAY       0000000000007d58  00006d58
       0000000000000008  0000000000000008  WA       0     0     8
  [25] .rela.fini_array  RELA             0000000000000000  00009e38
       0000000000000018  0000000000000018   I      36    24     8
  [26] .data.rel.ro      PROGBITS         0000000000007d60  00006d60
       0000000000000080  0000000000000000  WA       0     0     16
  [27] .rela.data.rel.ro RELA             0000000000000000  00009e50
       0000000000000180  0000000000000018   I      36    26     8
  [28] .dynamic          DYNAMIC          0000000000007de0  00006de0
       00000000000001f0  0000000000000010  WA       5     0     8
  [29] .got              PROGBITS         0000000000007fd0  00006fd0
       0000000000000028  0000000000000008  WA       0     0     8
  [30] .got.plt          PROGBITS         0000000000008000  00007000
       0000000000000050  0000000000000008  WA       0     0     8
  [31] .data             PROGBITS         0000000000008050  00007050
       000000000000002c  0000000000000000  WA       0     0     16
  [32] .rela.data        RELA             0000000000000000  00009fd0
       0000000000000060  0000000000000018   I      36    31     8
  [33] .tm_clone_table   PROGBITS         0000000000008080  00007080
       0000000000000000  0000000000000000  WA       0     0     8
  [34] .bss              NOBITS           0000000000008080  00007080
       0000000000000028  0000000000000000  WA       0     0     8
  [35] .comment          PROGBITS         0000000000000000  00007080
       0000000000000094  0000000000000001  MS       0     0     1
  [36] .symtab           SYMTAB           0000000000000000  00007118
       0000000000000f30  0000000000000018          37    95     8
  [37] .strtab           STRTAB           0000000000000000  00008048
       00000000000006f4  0000000000000000           0     0     1
  [38] .shstrtab         STRTAB           0000000000000000  0000a030
       0000000000000139  0000000000000000           0     0     1
```
这里是不包含.rela.text section的。下面用llvm-bolt对coremark.exe进行插桩。
```shell
~/project/llvm/build/bin/llvm-bolt coremark.exe --instrumentation-file=/home/wzl/project/coremark/perf.fdata  -instrument -o coremark.instr
```
会发现，bolt执行报错了：
```shell
BOLT-INFO: shared object or position-independent executable detected
BOLT-INFO: Target architecture: x86_64
BOLT-INFO: BOLT version: 5ff27fe1ff03d5aeaf8567c97618170f0cef8f58
BOLT-INFO: first alloc address is 0x0
BOLT-INFO: creating new program header table at address 0x200000, offset 0x200000
BOLT-ERROR: instrumentation runtime libraries require relocations
```
修改bolt源码：
```shell
diff --git a/bolt/include/bolt/Core/BinaryContext.h b/bolt/include/bolt/Core/BinaryContext.h
index ccd183369954..eafe0bf74a9c 100644
--- a/bolt/include/bolt/Core/BinaryContext.h
+++ b/bolt/include/bolt/Core/BinaryContext.h
@@ -552,7 +552,7 @@ public:
   std::unique_ptr<MCAsmBackend> MAB;
 
   /// Indicates if relocations are available for usage.
-  bool HasRelocations{false};
+  bool HasRelocations{true};
 
   /// Is the binary always loaded at a fixed address. Shared objects and
   /// position-independent executables (PIEs) are examples of binaries that
diff --git a/bolt/lib/Rewrite/RewriteInstance.cpp b/bolt/lib/Rewrite/RewriteInstance.cpp
index 2d3f987af959..4798456d02fc 100644
--- a/bolt/lib/Rewrite/RewriteInstance.cpp
+++ b/bolt/lib/Rewrite/RewriteInstance.cpp
@@ -1588,7 +1588,7 @@ Error RewriteInstance::readSpecialSections() {
               "Use -update-debug-sections to keep it.\n";
   }
 
-  HasTextRelocations = (bool)BC->getUniqueSectionByName(".rela.text");
+  HasTextRelocations = true; // (bool)BC->getUniqueSectionByName(".rela.text");
   LSDASection = BC->getUniqueSectionByName(".gcc_except_table");
   EHFrameSection = BC->getUniqueSectionByName(".eh_frame");
   GOTPLTSection = BC->getUniqueSectionByName(".got.plt");
```
这里修改的目的是，虽然coremark.exe中没有rela.text section，但是在bolt程序中当做有，看下会导致哪些问题。
再次执行上述的插桩命令，会发现bolt执行完了。接下来，执行coremark.instr。查看perf.fdata，会发现里面基本没有记录。
![](Bolt中Relocation作用分析/perf_fail.png)

为了定位数据输出问题，这里做一组对照实验。编译时链接器加--emit-relocs选项。

- 生成带relocaton section的coremark_reloc.exe
  ```
  ~/project/llvm/build/bin/clang -O2 -Wl,--emit-relocs -Ilinux -Iposix -I. -DFLAGS_STR=\""-O2 -DPERFORMANCE_RUN=1  -lrt"\" -DITERATIONS=0 -DPERFORMANCE_RUN=1 core_list_join.c core_main.c core_matrix.c core_state.c core_util.c posix/core_portme.c -o ./coremark_reloc.exe -lrt
  ```
- 对coremark_reloc.exe插桩生成coremark_reloc.instr
  - 注意，这里需要指定数据的保存位置且要用绝对路径，否则可能无法找到perf.fdata文件。
  ```
  ~/project/llvm/build/bin/llvm-bolt coremark_reloc.exe --instrumentation-file=/home/wzl/project/coremark/perf.fdata  -instrument -o coremark_reloc.instr
  ```
- 分别对coremark.instr和coremark_reloc.instr进行反汇编，对比反汇编的结果。
  ```
  objdump -d coremark.instr
  objdump -d coremark_reloc.instr
  ```
对比结果如下：
![](Bolt中Relocation作用分析/instr_diff.png)

主要区别是_start函数中的有三条立即数的mov指令，而这三个立即数分别对应__libc_csu_fini，__libc_csu_init和main函数的地址。左边是coremar.instr的反汇编，三个立即数分别对应原text section的函数地址，右边是coremark_reloc.instr的反汇编，三个立即数对应新text section中的函数地址。

简单说，coremark.exe中不含rela.text section，对于_start函数中的三条mov指令，bolt没有办法识别这三个立即数是与三个符号绑定的，因此，在函数地址发生变化以后，也就不能做地址更新。而coremark_reloc.exe在插桩时，反汇编_start函数时，可以通过rela.text section读取到三条指令分别与三个符号绑定，反汇编过程会将指令的立即数替换为symbol，后面经重新汇编和链接以后三条指令会更新为新的函数地址。

综上，relocation section的主要作用为辅助函数反汇编，因为有些指令，仅从操作码上无法判断其是否有symbol引用。而bolt插桩/优化又可能导致symbol地址发生变化，若反汇编时缺乏symbol引用信息，会影响后面汇编和链接时指令的更新。

**注：**上面的实验使用的clang14进行编译。若使用clang15，则_start中的三条mov指令对应的位置被编译成三条lea，此时coremark即使不包含rela.text section，仍可正常做插桩和优化。原因是lea指令可以通过操作码获知该指令包含symbol引用。

[文中生成的相关文件](https://github.com/zhihaishibei/BlogFile/tree/master/BoltRelocation)
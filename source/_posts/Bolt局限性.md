---
title: Bolt局限性
date: 2022-07-20 20:26:04
categories: PGO
tags: [bolt,post link optimizer,pgo]
---
## Bolt现存问题
### 一、CodeSize膨胀
Bolt在优化二进制文件后，会保留原始的.text section和.eh_frame section，并将原文件中的相应section重命名为.bolt.orig.text section。优化后的代码放在新生成的text section中，这导致优化后的文件中实际上存在两份代码段。.eh_frame section 同理。

目前bolt官方没有关注优化后二进制文件CodeSize问题，但是对CodeSize敏感的设备上二进制文件使能bolt，删除旧的sections则很有必要。要完全删除旧的sections，这要求Bolt可以对所有函数完成反汇编，可以分析出所有函数对外部符号的引用。实际上，存在一些场景，会导致函数反汇编失败。比如手写汇编可以在函数A内部调用该函数内部的某处指令B，这在反汇编时会被认为是不正常的指令。
### 二、依赖Relocation Section
Bolt的一个优点是可以直接对库文件或可执行文件进行优化。但实际上，若二进制中不包含relocation section，那么Bolt很难体现出优化效果。这就要求在编译二进制过程时保留Relocation section。
### 三、Align信息丢失
在编译流程中，生成的汇编码中可以看到每个函数对应的Align。经过汇编链接以后，Align信息没有保存在最终的库文件中。这导致bolt反汇编时也不能获得Align信息。大多数情况下，这是没有问题的，但还是会存在对Align有要求的情况。比如虚拟机中常见的模板解释器是使用汇编实现的，这里对函数是有Align依赖的，函数的size必须要保持相同，不足的要用NOP补齐。若Bolt破坏掉这个Align规则，会导致解释器运行失败。当然，Bolt提供了skip-funcs的功能，即跳过指定的函数不处理。但这要求Bolt使用者对目标优化库文件有充分的认知，不然很可能会出现莫名其妙的问题。
### 四、Symbol地址更新存在不确定性
以下面的程序为例：
```
funcA:
...
funcA_end
funcB:
...
```
funcA_end代表funcA的结束位置，但地址与funcB的首地址一致。经bolt处理后，可能存在函数重排，funcA_end更新为funcA的结束位置还是funcB的首地址是存在不确定性的。
### 五、对DT_FINI的依赖
bolt处理动态库文件，要求动态库的.dynamic section包含DT_FINI entry，其作用是做钩子函数，bolt处理后会将改地址更新为新函数地址，新函数的功能包括：
- 保存profile数据
- 调用原来的fini函数

而DT_FINI entry并非.dynamic section的必选项。所以要求编译库文件过程中指定fini函数，如ld.lld可通过--fini=symbol指定。
### 六、GOT表更新
对于多入口函数，除主入口外，其他入口若在GOT表中有相应的entry，则函数位置变化以后，GOT表不能正常更新。
### 七、插桩后的库文件生成WE的segment
有些平台不支持可写可执行的segment，因此需要对WE segment进行拆分
### 八、对arm64 relocation的处理前后不一致
```c++
void RewriteInstance::readRelocations(const SectionRef &Section) {
  ...
      // In AArch64 there are zero reasons to keep a reference to the
      // "original" symbol plus addend. The original symbol is probably just a
      // section symbol. If we are here, this means we are probably accessing
      // data, so it is imperative to keep the original address.
      if (IsAArch64) {
        SymbolName = ("SYMBOLat0x" + Twine::utohexstr(Address)).str();
        SymbolAddress = Address;
        Addend = 0;
      }
  ...
}
```
```c++
bool RewriteInstance::analyzeRelocation(
    const RelocationRef &Rel, uint64_t RType, std::string &SymbolName,
    bool &IsSectionRelocation, uint64_t &SymbolAddress, int64_t &Addend,
    uint64_t &ExtractedValue, bool &Skip) const {
  ...
  if (SymbolIter == InputFile->symbol_end()) {
    SymbolAddress = ExtractedValue - Addend + PCRelOffset;
    MCSymbol *RelSymbol =
        BC->getOrCreateGlobalSymbol(SymbolAddress, "RELSYMat");
    SymbolName = std::string(RelSymbol->getName());
    IsSectionRelocation = false;
  } else {
    ...
  }
}
```
对arm64 relocation的处理前后不一致，导致找不到符号。
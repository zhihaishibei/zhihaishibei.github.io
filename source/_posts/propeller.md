---
title: Propeller
categories: compiler
tags: [propeller,post link optimizer,pgo]
---
# Propeller 源码分析
## 一、源码下载
```
git clone https://github.com/google/llvm-propeller.git
```
下载后默认分支是bb-clusters，该分支实际上是不包含propeller优化的。切换分支到plo-dev，就可以查看相关源码了。
```
git checkout -b plo-dev origin/plo-dev
```
## 二、源码解析
lld/ELF/Driver.cpp
```c++
// Do actual linking. Note that when this function is called,
// all linker scripts have already been parsed.
template <class ELFT> void LinkerDriver::link(opt::InputArgList &args) {
  llvm::TimeTraceScope timeScope("Link", StringRef("LinkerDriver::Link"));
  // If a -hash-style option was not given, set to a default value,
  // which varies depending on the target.
  if (!args.hasArg(OPT_hash_style)) {
    if (config->emachine == EM_MIPS)
      config->sysvHash = true;
    else
      config->sysvHash = config->gnuHash = true;
  }

  // Default output filename is "a.out" by the Unix tradition.
  if (config->outputFile.empty())
    config->outputFile = "a.out";

  ...

  lld::propeller::doPropeller();

  ...

  // Write the result to the file.
  writeResult<ELFT>();
}
```
在link中调用`doPropeller`，该函数就是Propeller优化器的入口。

lld/ELF/LinkerPropeller.cpp
```c++
// Propeller framework entrance.
void doPropeller() {
  ...
  setupConfig();
  prop = make<Propeller>();

  if (!prop->maybeCheckTarget()) {
    warn("[Propeller]: Propeller skipped '" + config->outputFile + "'.");
    return;
  }

  std::vector<ObjectView *> objectViews;
  std::for_each(objectFiles.begin(), objectFiles.end(),
                 [&objectViews](const InputFile *inf) {
                   auto *ov = Propeller::createObjectView(
                       inf->getName(), objectViews.size() + 1, inf->mb);
                   if (ov)
                     objectViews.push_back(ov);
                 });
  if (prop->processFiles(objectViews))
    config->symbolOrderingFile = prop->genSymbolOrderingFile();
  else
    error("Propeller stage failed.");
}
```
函数执行`processFiles` 和 `genSymbolOrderingFile`。
### 2.1 processFiles执行过程
lld/ELF/Propeller/Propeller.cpp
```c++
// Entrance of Propeller framework. This processes each elf input file in
// parallel and stores the result information.
bool Propeller::processFiles(std::vector<ObjectView *> &views) {
  if (!propf->readSymbols()) {
    error(std::string("invalid propfile: '") + propConfig.optPropeller.str() +
          "'");
    return false;
  }

  processFailureCount = 0;
  llvm::parallel::for_each(
      llvm::parallel::parallel_execution_policy(), views.begin(), views.end(),
      std::bind(&Propeller::processFile, this, std::placeholders::_1));

  if (processFailureCount * 100 / views.size() >= 50)
    warn("propeller failed to parse more than half the objects, "
         "optimization would suffer");

  /* Drop alias cfgs. */
  ...

  // Map profiles.
  if (!propf->processProfile())
    return false;

  calculateNodeFreqs();

  dumpCfgs();

  // Releasing all support data (symbol ordinal / name map, saved string refs,
  // etc) before moving to reordering.
  propf.reset(nullptr);
  return true;
}
```
```c++
void Propeller::processFile(ObjectView *view) {
  if (view) {
    std::map<uint64_t, uint64_t> ordinalRemapping;
    if (CFGBuilder(view).buildCFGs(ordinalRemapping)) {
      // Updating global data structure.
      std::lock_guard<std::mutex> lockGuard(lock);
      Views.emplace_back(view);
      for (std::pair<const StringRef, std::unique_ptr<ControlFlowGraph>> &p :
           view->cfgs) {
        auto result = cfgMap[p.first].emplace(p.second.get());
        (void)(result);
        assert(result.second);
      }
      propf->ordinalRemapping.insert(ordinalRemapping.begin(),
                                     ordinalRemapping.end());

    } else {
      warn("skipped building controlFlowGraph for '" + view->viewName + "'");
      ++processFailureCount;
    }
  }
}
```
在processFiles函数中，流程总结如下：
- 遍历所有输入的ELF文件，为每个ELF文件，为其包含的所有函数构建CFG
- 读取并映射profile信息
- 计算执行频率。


### 2.2 genSymbolOrderingFile 执行过程
```c++
// Generate symbol ordering file according to selected optimization pass and
// feed it to the linker.
std::vector<StringRef> Propeller::genSymbolOrderingFile() {
  ...

  std::list<StringRef> symbolList(1, "hot");
  const auto hotPlaceHolder = symbolList.begin();
  const auto coldPlaceHolder = symbolList.end();
  propLayout = make<CodeLayout>();
  propLayout->doSplitOrder(symbolList, hotPlaceHolder, coldPlaceHolder);
  ...

  calculateLegacy(symbolList, hotPlaceHolder, coldPlaceHolder);

  ...

  symbolList.erase(hotPlaceHolder);

  return std::vector<StringRef>(
      std::move_iterator<std::list<StringRef>::iterator>(symbolList.begin()),
      std::move_iterator<std::list<StringRef>::iterator>(symbolList.end()));
}
```
核心函数是doSplitOrder，该函数为CodeLayout的入口函数。

lld/ELF/Propeller/CodeLayout/CodeLayout.cpp

```C++
// This function iterates over the cfgs included in the Propeller profile and
// adds them to cold and hot cfg lists. Then it appropriately performs basic
// block reordering by calling NodeChainBuilder.doOrder() either on all cfgs (if
// -propeller-opt=reorder-ip) or individually on every controlFlowGraph. After
// creating all the node chains, it hands the basic block chains to a
// ChainClustering instance for further rerodering.
void CodeLayout::doSplitOrder(std::list<StringRef> &symbolList,
                              std::list<StringRef>::iterator hotPlaceHolder,
                              std::list<StringRef>::iterator coldPlaceHolder) {
  std::chrono::steady_clock::time_point start =
      std::chrono::steady_clock::now();

  // Populate the hot and cold cfg lists by iterating over the cfgs in the
  // propeller profile.
  prop->forEachCfgRef([this](ControlFlowGraph &cfg) {
    if (cfg.isHot()) {
      hotCFGs.push_back(&cfg);
      if (propConfig.optPrintStats) {
        // Dump the number of basic blocks and hot basic blocks for every
        // function
        unsigned hotBBs = 0;
        unsigned allBBs = 0;
        cfg.forEachNodeRef([&hotBBs, &allBBs](CFGNode &node) {
          if (node.freq)
            hotBBs++;
          allBBs++;
        });
        fprintf(stderr, "HISTOGRAM: %s,%u,%u\n", cfg.name.str().c_str(), allBBs,
                hotBBs);
      }
    } else
      coldCFGs.push_back(&cfg);
  });


  if (propConfig.optReorderBlocksRandom)
    clustering.reset(new NoOrdering());
  if (propConfig.optReorderIP || propConfig.optReorderFuncs)
    clustering.reset(new CallChainClustering());
  else {
    // If function ordering is disabled, we want to conform the the initial
    // ordering of functions in both the hot and the cold layout.
    clustering.reset(new NoOrdering());
  }

  std::srand(unsigned(std::time(0)));

  if (propConfig.optReorderIP) {
    // If -propeller-opt=reorder-ip we want to run basic block reordering on all
    // the basic blocks of the hot cfgs.
    NodeChainBuilder(hotCFGs).doOrder(clustering);
  } else if (propConfig.optReorderBlocks) {
    // Otherwise we apply reordering on every controlFlowGraph separately
    for (ControlFlowGraph *cfg : hotCFGs)
      NodeChainBuilder(cfg).doOrder(clustering);
  } else {
    // If reordering is not desired, we create changes according to the initial
    // order in the controlFlowGraph.
    for (ControlFlowGraph *cfg : hotCFGs) {
      if (propConfig.optSplitFuncs) {
        std::vector<CFGNode *> coldNodes, hotNodes;
        cfg->forEachNodeRef([&coldNodes, &hotNodes](CFGNode &n) {
          if (n.freq)
            hotNodes.emplace_back(&n);
          else
            coldNodes.emplace_back(&n);
        });
        auto compareNodeAddress = [](CFGNode *n1, CFGNode *n2) {
          return n1->mappedAddr < n2->mappedAddr;
        };
        sort(hotNodes.begin(), hotNodes.end(), compareNodeAddress);
        sort(coldNodes.begin(), coldNodes.end(), compareNodeAddress);
        if (propConfig.optReorderBlocksRandom && hotNodes.size() > 1) {
          std::srand(unsigned(std::time(0)));
          std::random_shuffle(hotNodes.begin(), hotNodes.end());
        }
        if (!hotNodes.empty())
          clustering->addChain(std::make_unique<NodeChain>(hotNodes));
        if (!coldNodes.empty())
          clustering->addChain(std::make_unique<NodeChain>(coldNodes));
      } else {
	// No split functions.
	if (propConfig.optReorderBlocksRandom) {
	  std::vector<CFGNode *> randomized_nodes;
	  cfg->forEachNodeRef([&randomized_nodes](CFGNode &n) {
				randomized_nodes.emplace_back(&n);
			      });
          std::srand(unsigned(std::time(0)));
	  std::random_shuffle(randomized_nodes.begin(), randomized_nodes.end());
	  clustering->addChain(std::make_unique<NodeChain>(randomized_nodes));
	} else {
	  clustering->addChain(std::make_unique<NodeChain>(cfg));
	}
      }
    }
    warn("[Propeller]: generated randomized bb ordering for all hot functions.");
  }

  // The order for cold cfgs remains unchanged.
  for (ControlFlowGraph *cfg : coldCFGs)
    clustering->addChain(std::make_unique<NodeChain>(cfg));

  // After building all the chains, let the chain clustering algorithm perform
  // the final reordering and populate the hot and cold cfg node orders.
  clustering->doOrder(HotOrder, ColdOrder);

  // Transfter the order to the symbol list.
  for (CFGNode *n : HotOrder)
    symbolList.insert(hotPlaceHolder, n->shName);

  for (CFGNode *n : ColdOrder)
    symbolList.insert(coldPlaceHolder, n->shName);

  std::chrono::steady_clock::time_point end = std::chrono::steady_clock::now();
  warn("[Propeller]: bb reordering took: " +
       Twine(std::chrono::duration_cast<std::chrono::milliseconds>(end - start)
                 .count()));

  if (propConfig.optPrintStats)
    printStats();
}
```

分析源码，基本上明确如下几个问题：

1. propeller是link的一个执行步骤
2. profile数据的读取是在linker中
3. 优化主要体现在依据计算出的执行频率，重排代码布局

后续将分析LTO的执行过程，并比较LTO与propeller的区别。
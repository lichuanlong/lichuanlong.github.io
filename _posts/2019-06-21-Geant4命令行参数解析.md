---
layout: post
title:  "Geant4命令行参数解析"
date:   2019-06-22
categories: Geant4
typora-root-url: ..
---

# 需求

> 改一次参数，重新编译一次Geant4应用程序，实在是弱爆了。

当开始跑程序了，就会有上面的想法了。主要原因是我们得做一个无人值守的运行过程。跑一次编译一次那不是总得盯着程序运行完了没有么？

编好脚本，敲下enter键，我们去喝杯咖啡聊聊天，或者明天后天再来接收计算结果，多么的惬意啊。

像这种功能需求在实际工作中还是挺多的。比如我想考察屏蔽材料不同厚度的效果，起码得做几个厚度计算一下画一条曲线。再比如我想考察多球中子谱仪不同厚度的聚乙烯球壳对响应的影响，也得算好几次，且不说单能中子能量算的次数更多。

总结起来，主要是探测器构建类的材料，几何等方面需要做多次改动多次计算。也可以统称为方案。

# 解决思路

总体思路是得把参数传进去。编译一次复用。

- 从命令行参数传进去。
- 从宏文件设置命令把参数传进去。

第一个是最近在想的办法，这里就说这个。解析命令行参数是个很基本的事，用Geant4的人估计天天用命令行终端。比如`ls`命令，带上`-l`参数是用长格式显示列表。编写ls这个程序的人就得解析后面的各种参数。

还是从Geant4的例子里~~找灵感~~，哦不，是copy。

B4a例子头一次把程序参数做得这么规矩且丰富，我是从B1看过来的，所以说头一次。其实每一个例子都要判断一下是交互模式运行还是batch模式运行，也是一个简单的解析命令行参数的过程。

看下面B4a程序的使用说明，可以设置macro，可以设置线程数，对于多线程的程序。还可以设置ui？算了不管他，我们都用qt对不对。

```c++
//....oooOO0OOooo........oooOO0OOooo........oooOO0OOooo........oooOO0OOooo......

namespace {
  void PrintUsage() {
    G4cerr << " Usage: " << G4endl;
    G4cerr << " exampleB4a [-m macro ] [-u UIsession] [-t nThreads]" << G4endl;
    G4cerr << "   note: -t option is available only for multi-threaded mode."
           << G4endl;
  }
}

//....oooOO0OOooo........oooOO0OOooo........oooOO0OOooo........oooOO0OOooo......
```

至于怎么实现的，读读B4a的main函数吧。

```c++
//....oooOO0OOooo........oooOO0OOooo........oooOO0OOooo........oooOO0OOooo......

int main(int argc,char** argv)
{
  // Evaluate arguments
  //
  if ( argc > 7 ) {
    PrintUsage();
    return 1;
  }
  
  G4String macro;
  G4String session;
#ifdef G4MULTITHREADED
  G4int nThreads = 0;
#endif
  for ( G4int i=1; i<argc; i=i+2 ) {
    if      ( G4String(argv[i]) == "-m" ) macro = argv[i+1];
    else if ( G4String(argv[i]) == "-u" ) session = argv[i+1];
#ifdef G4MULTITHREADED
    else if ( G4String(argv[i]) == "-t" ) {
      nThreads = G4UIcommand::ConvertToInt(argv[i+1]);
    }
#endif
    else {
      PrintUsage();
      return 1;
    }
  }  
  
  // Detect interactive mode (if no macro provided) and define UI session
  //
  G4UIExecutive* ui = nullptr;
  if ( ! macro.size() ) {
    ui = new G4UIExecutive(argc, argv, session);
  }

  // Choose the Random engine
  //
  G4Random::setTheEngine(new CLHEP::RanecuEngine);
  
  // Construct the default run manager
  //
#ifdef G4MULTITHREADED
  auto runManager = new G4MTRunManager;
  if ( nThreads > 0 ) { 
    runManager->SetNumberOfThreads(nThreads);
  }  
#else
  auto runManager = new G4RunManager;
#endif

  // Set mandatory initialization classes
  //
  auto detConstruction = new B4DetectorConstruction();
  runManager->SetUserInitialization(detConstruction);

  auto physicsList = new FTFP_BERT;
  runManager->SetUserInitialization(physicsList);
    
  auto actionInitialization = new B4aActionInitialization(detConstruction);
  runManager->SetUserInitialization(actionInitialization);
  
  // Initialize visualization
  //
  auto visManager = new G4VisExecutive;
  // G4VisExecutive can take a verbosity argument - see /vis/verbose guidance.
  // G4VisManager* visManager = new G4VisExecutive("Quiet");
  visManager->Initialize();

  // Get the pointer to the User Interface manager
  auto UImanager = G4UImanager::GetUIpointer();

  // Process macro or start UI session
  //
  if ( macro.size() ) {
    // batch mode
    G4String command = "/control/execute ";
    UImanager->ApplyCommand(command+macro);
  }
  else  {  
    // interactive mode : define UI session
    UImanager->ApplyCommand("/control/execute init_vis.mac");
    if (ui->IsGUI()) {
      UImanager->ApplyCommand("/control/execute gui.mac");
    }
    ui->SessionStart();
    delete ui;
  }

  // Job termination
  // Free the store: user actions, physics_list and detector_description are
  // owned and deleted by the run manager, so they should not be deleted 
  // in the main() program !

  delete visManager;
  delete runManager;
}

//....oooOO0OOooo........oooOO0OOooo........oooOO0OOooo........oooOO0OOooo.....

```



# 怎么把参数传给探测器构建类

让我在例子里再找找

<http://www-geant4.kek.jp/lxr/source/examples/extended/common/src/DetectorConstruction0.cc#L44>

common这个例子的DetectorConstruction0类的构造函数就跟basic例子很不一样，带参数。

```c++
  public:
    DetectorConstruction0(
       const G4String& materialName = "G4_AIR",
       G4double hx = 50*CLHEP::cm, 
       G4double hy = 50*CLHEP::cm, 
       G4double hz = 50*CLHEP::cm);
```

看到这个例子应该知道怎么模仿了，把从命令行解析出来的参数传给这个带参数的构造函数，在初始化DetectorConstruction0时。

虽然common例子没有这么做，用的是默认参数值。

```c++
  // Instantiate all detector construction classes
  auto detectorConstruction = new DetectorConstruction;
  auto detectorConstruction0 = new DetectorConstruction0;
```

# 看懂了，那试一试吧

不用说了，一定得改B4a例子啊。改几何是最明显的了。

例子建了个10层的模型

![](/image/layer10.png)

参数是写死了的。

```c++
  // Geometry parameters
  G4int nofLayers = 10;
```

我们把它改成从命令行参数传进去，比如5层。

可视化用命令`./exampleB4a -n 5`，添加一个参数设置层数。

[修改B4a例子的代码包](/zip/B4a_Geant4.10.5.p01.zip)基于Geant4.10.05.p01的B4a例子，请自行比较改动。

![](/image/layer5.png)


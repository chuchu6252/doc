# Carla 中的 Chrono 实现

这是一个 UE4 车辆运动组件，用 Chrono 物理引擎（专业多体动力学库）替代 UE4 的 PhysX，提供更精确的车辆动力学、悬挂、轮胎和地形交互模拟。

Chrono 需要在虚幻引擎端启用 RTTI（RTTI（Run-Time Type Information），从而支持动态类型识别和异常处理等功能。
文件 [`Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Carla.Build.cs`](https://github.com/OpenHUTB/carla/blob/1b453d00cbf5e0afa00f3e993799c36c9286d75c/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Carla.Build.cs#L214) 中增加`bUseRTTI = true;`。

!!! 笔记
    每个.Build.cs文件声明派生自ModuleRules基类的类，并设置属性控制器从构造函数进行编译的方式。由虚幻编译工具编译，并被构造来确定整体编译环境。使用C#语法。

## 流程

1. 初始化阶段 ([CreateChronoMovementComponent](https://github.com/OpenHUTB/hutb/blob/f96c0e068a023d01fcf761894b142d09ab42ab84/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Vehicle/MovementComponents/ChronoMovementComponent.cpp#L21) + [BeginPlay](https://github.com/OpenHUTB/hutb/blob/f96c0e068a023d01fcf761894b142d09ab42ab84/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Vehicle/MovementComponents/ChronoMovementComponent.cpp#L142) )

    1.1 [创建组件并配置 JSON 参数（车辆、动力总成、轮胎配置）](https://github.com/OpenHUTB/hutb/blob/f96c0e068a023d01fcf761894b142d09ab42ab84/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Vehicle/MovementComponents/ChronoMovementComponent.cpp#L31)
    
    1.2 [BeginPlay：初始化 Chrono 系统](https://github.com/OpenHUTB/hutb/blob/f96c0e068a023d01fcf761894b142d09ab42ab84/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Vehicle/MovementComponents/ChronoMovementComponent.cpp#L142) ：
    设置重力、求解器（Barzilai -Borwein 方法是一种迭代梯度下降法）、迭代次数
    
    1.3 [InitializeChronoVehicle：从 JSON 文件加载车辆模型（底盘、悬挂、动力系统、轮胎）](https://github.com/OpenHUTB/hutb/blob/f96c0e068a023d01fcf761894b142d09ab42ab84/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Vehicle/MovementComponents/ChronoMovementComponent.cpp#L167)
    
    1.4. [创建地形交互层（光线投射到 UE4 碰撞）](https://github.com/OpenHUTB/hutb/blob/f96c0e068a023d01fcf761894b142d09ab42ab84/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Vehicle/MovementComponents/ChronoMovementComponent.cpp#L157)

2. Carla -> Chrono：每帧更新 ( [TickComponent](https://github.com/OpenHUTB/hutb/blob/f96c0e068a023d01fcf761894b142d09ab42ab84/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Vehicle/MovementComponents/ChronoMovementComponent.cpp#L235) )

    2.1 时间子步处理：

    如果单帧时间 > MaxSubstepDeltaTime（默认 0.01s），分解为多个小步进行精确计算。否则直接用该帧时间步

    2.2 物理仿真：[AdvanceChronoSimulation(StepSize)](https://github.com/OpenHUTB/hutb/blob/f96c0e068a023d01fcf761894b142d09ab42ab84/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Vehicle/MovementComponents/ChronoMovementComponent.cpp#L292) ：同步时间、控制（转向、油门、刹车）、地形 到 chrono


3. Chrono -> Carla：[位置/旋转同步回 UE4](https://github.com/OpenHUTB/hutb/blob/f96c0e068a023d01fcf761894b142d09ab42ab84/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Vehicle/MovementComponents/ChronoMovementComponent.cpp#L275) ：

    坐标系转换：Chrono（右手系，米）↔ UE4（左手系，厘米）

    地形交互 (UERayCastTerrain)：Chrono 需要知道地形 [高度](https://github.com/OpenHUTB/hutb/blob/f96c0e068a023d01fcf761894b142d09ab42ab84/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Vehicle/MovementComponents/ChronoMovementComponent.cpp#L114) 、[法线](https://github.com/OpenHUTB/hutb/blob/f96c0e068a023d01fcf761894b142d09ab42ab84/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Vehicle/MovementComponents/ChronoMovementComponent.cpp#L125) 、[摩擦系数](https://github.com/OpenHUTB/hutb/blob/f96c0e068a023d01fcf761894b142d09ab42ab84/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Vehicle/MovementComponents/ChronoMovementComponent.cpp#L137) 来计算轮胎力。本实现通过光线投射到 UE4


## 提交历史

在 [commit页面](https://github.com/OpenHUTB/hutb/commits/hutb/?author=Axel1092&after=3225f8cfb1920f4ebf8a98a658385b379a04541b+174) 按用户名 [`Axel1092`](https://github.com/Axel1092) 过滤。


* Support custom chrono (#9182)
* lower COM Z-height to 0.52 m for realistic sedan dynamics (#9126)
* Removed harcoded path for chrono(#9100)
* generate all supported PythonAPI installation file (#3260)
* fix "carla\Unreal\CarlaUE4\Plugins\Carla\CarlaDependencies\dll\ChronoEngine.dll" missing caused by chrono source code download failed
* 兼容VS2019和VS2022 (#3227)
* Added restore_physx_physics function
* Add build support for VS2022 and Ninja for LibCarla and osm2odr on Windows
* Make (Unreal) non-unity build available (#6190)
* Changed rotation and position offsets for chrono. Disabled overlap events for some meshes
* Changed Custom movement component begin play execution order to prevent disable before initialization
* Adding new attachment type 'SpringArmGhost'
* Luis/collisions bushes (#5477)
* Fixed issues with new map.name
* Re-Upload : Enabled overlap events on static meshes for chrono-carsim
* Setup Visual Studio 2019 for windows
* Fix minor typo
* [Large Maps (#4207)](https://github.com/OpenHUTB/hutb/commit/c46c65ff4f3f904f575dbd19078d4a690c616f43)
* [Corkyw10/chrono docs (#4160)]() - chrono 文档
* [Disable chrono upon collision](https://github.com/OpenHUTB/hutb/commit/3679aaa2af46a00c1e92a44da55513d0c01812cb) - 当碰撞时候禁用 chrono
* [Added collision detection for chrono vehicles.](https://github.com/OpenHUTB/hutb/commit/5da83fbfb68ab1de10b07368bed996ce8438cc8c) - 给启用 chrono 的载具添加[碰撞检测](https://www.projectchrono.org/assets/slides_3_0_0/3_Contact/2_Chrono_CollisionDetection.pdf)器
* [Added trace profiler events for unreal insights.](https://github.com/OpenHUTB/hutb/commit/2584d884b971e49a35b334b06dce92d930091366)
* [Fixed chrono Dlls separated from the binary file](https://github.com/OpenHUTB/hutb/commit/55a98646b3af678d2965fb391a42e84c902cc454)
* [Removed tabs.](https://github.com/OpenHUTB/hutb/commit/226942381dc9bcbdd6754ab969861e07d4ae0d54)
* [Added missing chrono macros](https://github.com/OpenHUTB/hutb/commit/74180b0fe7e6e2d552b50fac145444e916df9056)
* [Fixed std issues in chrono implementation in windows.](https://github.com/OpenHUTB/hutb/commit/bffd3968a4c510dddc7d94704a13d78ae860d3a9)
* [Enabling exceptions for chrono package.](https://github.com/OpenHUTB/hutb/commit/a1704cb3c74db4b15c8cf17b0dd7321afc43d247)
* [Overloaded all UBaseCarlaMovementComponent functions in UChronoMovementComponent](https://github.com/OpenHUTB/hutb/commit/c5a1a28fa20593ccd5105b53f80ab14fa759327a)
* [Added JSON ingestion for chrono vehicle definition](https://github.com/OpenHUTB/hutb/commit/3fc6bce36ccc61cc8ebeea3c673af2047cdfc506)
* [Review fixes.](https://github.com/OpenHUTB/hutb/commit/a93376e2026f5abc7ae8996c7e0f4d1cde51c6a1)
* [Set chrono version to 6.0.0.](https://github.com/OpenHUTB/hutb/commit/546016c664ef1c84d216253c8e38e083f2040168)
* [Skipping 0 delta times.](https://github.com/OpenHUTB/hutb/commit/d6d4b1f50d0f868707ca5a1092bb4c1256bf9c48)
* [Small fix for Windows compilation without chrono.](https://github.com/OpenHUTB/hutb/commit/723d23aaeddd2c2896c43fd9cf2dece1c23debc3)
* [Fixed fall through bug for chrono vehicle.](https://github.com/OpenHUTB/hutb/commit/3c1f515b3fc556588d0d452678494020398b08cb)
* [Small fix to windows build](https://github.com/OpenHUTB/hutb/commit/aef432e08bb2ec99c43dd521342709c66544f705)
* [Added missing .lib from chrono to Build.cs](https://github.com/OpenHUTB/hutb/commit/0473b18df1c59668298896d4037ba3d7c90571a7)
* [Updated chrono install for windows.](https://github.com/OpenHUTB/hutb/commit/028e029404af2729622891f24e4a5f8dc4112e57)
* [Added release compile mode for chrono library.](https://github.com/OpenHUTB/hutb/commit/bf8e160622225e5fc8fea086aa2414bb662026ff)
* [Added substep algorithm for chrono. Added initial UE4 terrain detection with raycast.](https://github.com/OpenHUTB/hutb/commit/f45e0d7c08b6db2bd68e3c656bb850e4406f3f99)
* [Added chrono compilation support for windows.](https://github.com/OpenHUTB/hutb/commit/14a634aa29f6ad66df3584a441adf00d9e019fa3)
* [Created base chrono implementation](https://github.com/OpenHUTB/hutb/commit/2117794dd7989b276317743b99430468628f5baa)
* [Added chrono library](https://github.com/OpenHUTB/hutb/commit/7c3367325fad6310202d40490081f946b5f363c9)



## 参考

* [CARLA0.9.16与Chrono联合仿真--源代码解读](https://www.bilibili.com/video/BV1hb1FBYEGc/)
* [Unreal 插件](https://www.jianshu.com/p/e41a810b10ca)
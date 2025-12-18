# Carla、虚拟现实、无人机模式之间的切换

Carla 模式主要关注行人和自动驾驶车辆的场景模拟，VR 模式针对智能驾驶舱的行为和交互虚拟现实模拟，AIR 模式重点解决低空无人机的模拟，该页面的文档主要解决它们三者之间的切换问题。



## 使用方法

下载 [链接](https://pan.baidu.com/s/1n2fJvWff4pbtMe97GOqtvQ?pwd=hutb) 中的`software/hutb/hutb_car_vr_air_mujoco.zip`，`CarlaUE4.exe`启动场景的默认游戏模式为 Carla。
使用 [config.py](https://github.com/OpenHUTB/hutb/blob/hutb/PythonAPI/util/config.py) 切换到 [VR 模式](../interbehavior.md) ：
```shell
config.py --map Town10HD?GAME=VR
```
![](../img/pedestrian/VR_demo.gif)

切换到 [无人机模式](https://openhutb.github.io/air_doc/) ：
```shell
config.py --map Town10HD?GAME=AIR
```
参数中的`Town10HD`为所要切换的地图名，`GAME=`后面为所需要切到到的游戏模式名，目前支持：CARLA、VR、AIR 三种模式。

![](https://openhutb.github.io/air_doc/images/dev/HUTB_simulation.gif)


## 切换地图的实现分析

![HUTB_Modules](../img/build_modules.jpg)

调用 config.py 切换地图时 Carla 模式的 `--map` 参数不支持游戏模式的切换，为了支持游戏模式的切换，先分析已有实现的调用过程：

1. 脚本 config.py 使用  `load_world()` 函数实现世界的加载，然后调用`LoadWorld`，该功能的实现位于 **PythonAPI** 中的客户端 [PythonAPI\carla\source\libcarla\Client.cpp](https://github.com/OpenHUTB/hutb/blob/hutb/PythonAPI/carla/source/libcarla/Client.cpp#L227)
```cpp
.def("load_world", CONST_CALL_WITHOUT_GIL_3(cc::Client, LoadWorld, std::string, bool, rpc::MapLayer), (arg("map_name"), arg("reset_settings")=true, arg("map_layers")=rpc::MapLayer::All))
```

2. **客户端** 的 LoadWorld 实现位于 [LibCarla\source\carla\client\Client.h](https://github.com/OpenHUTB/hutb/blob/f32283bd214907196de92f98b0e7ffe583b45bf4/LibCarla/source/carla/client/Client.h#L75) ，

```cpp
World LoadWorld(
    std::string map_name,
    bool reset_settings = true,
    rpc::MapLayer map_layers = rpc::MapLayer::All) const {
  return World{_simulator->LoadEpisode(std::move(map_name), reset_settings, map_layers)};
}
```

再调用了 LibCarla client\detail 的模拟器 [LibCarla\source\carla\client\detail\Simulator.cpp](https://github.com/OpenHUTB/hutb/blob/f32283bd214907196de92f98b0e7ffe583b45bf4/LibCarla/source/carla/client/detail/Simulator.cpp#L87) 的 LoadEpisode 

```cpp
EpisodeProxy Simulator::LoadEpisode(std::string map_name, bool reset_settings, rpc::MapLayer map_layers) {
  ...
  _client.LoadEpisode(std::move(map_name), reset_settings, map_layers);
```

然后调用了 LibCarla client\detail 的客户端 [LibCarla\source\carla\client\detail\Client.cpp](https://github.com/OpenHUTB/hutb/blob/f32283bd214907196de92f98b0e7ffe583b45bf4/LibCarla/source/carla/client/detail/Client.cpp#L147)

```cpp
void Client::LoadEpisode(std::string map_name, bool reset_settings, rpc::MapLayer map_layer) {
  _pimpl->CallAndWait<void>("load_new_episode", std::move(map_name), reset_settings, map_layer);
}
```
最后异步调用服务端的 `load_new_episode`。

3. **服务端** 的 load_new_episode 位于 Server 模块的 Carla 服务器 [Unreal\CarlaUE4\Plugins\Carla\Source\Carla\Server\CarlaServer.cpp](https://github.com/OpenHUTB/hutb/blob/f32283bd214907196de92f98b0e7ffe583b45bf4/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Server/CarlaServer.cpp#L319) ，

```cpp
BIND_SYNC(load_new_episode) << [this](const std::string &map_name, const bool reset_settings, cr::MapLayer MapLayers) -> R<void>
{
  ...
  if(!Episode->LoadNewEpisode(cr::ToFString(map_name), reset_settings))
  ...
};
```
调用了 Game 模块中 轮次的 LoadNewEpisode 函数（位于 [Unreal\CarlaUE4\Plugins\Carla\Source\Carla\Game\CarlaEpisode.cpp](https://github.com/OpenHUTB/hutb/blob/f32283bd214907196de92f98b0e7ffe583b45bf4/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Game/CarlaEpisode.cpp#L122) ） 

```cpp
bool UCarlaEpisode::LoadNewEpisode(const FString &MapString, bool ResetSettings)
{
    ...
    UGameplayStatics::OpenLevel(GetWorld(), *FinalPath, true);
    ...
```

由此可以看出切换地图调用了模拟引擎的 `UGameplayStatics::OpenLevel()`。 


## 增加动态切换游戏模式

一般来讲，设置游戏模式（GameMode）都是在 WorldSettings 里面。
然而，该方法都只能在编辑器内提前配置好将要使用的 GameMode，无法在游戏时动态决定 GameMode。
下面的方法可以在运行游戏时决定 Map 使用哪一个 GameMode。

1. 在 [Unreal\CarlaUE4\Config\DefaultEngine.ini](https://github.com/OpenHUTB/hutb/blob/f32283bd214907196de92f98b0e7ffe583b45bf4/Unreal/CarlaUE4/Config/DefaultEngine.ini#L29) 中设置游戏模式类的别名
2. 调用 `UGameplayStatics::OpenLevel()` 函数，传入Options参数"Game=XXX"，或者直接在LevelName里面拼接好"LevelName?Game=XXX"，其中XXX是GameMode别名

```ini
+GameModeClassAliases=(Name="CARLA",GameMode="/Game/Carla/Blueprints/Game/CarlaGameMode.CarlaGameMode_C")
+GameModeClassAliases=(Name="VR",GameMode="/Script/CarlaUE4.DReyeVRGameMode")
+GameModeClassAliases=(Name="AIR",GameMode="/Script/AirSim.AirSimGameMode")
```



## 参考

* [绕开 WorldSetting 切换 GameMode 的详细分析](https://zhuanlan.zhihu.com/p/660795062?share_code=1kQREJgxnStpH&utm_psn=1953537310399370643)

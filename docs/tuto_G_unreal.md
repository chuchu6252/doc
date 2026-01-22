# 虚幻引擎高级特性

## [控制台命令](https://blog.csdn.net/somnusand/article/details/115511383) 

启动模拟后，输入`~`键调用控制台命令。

`stat rhi`: Rendering Hardware Interface，显示RHI内存和性能统计数据。

`HighResShot 3840x2160` —— 指定分辨率截图（分辨率），图片存储位置：`\Saved\Screenshots\`。


## [渲染管线](https://zhuanlan.zhihu.com/p/373995698) 

1. 动画模拟
2. 光线传播
3. 几何表示
4. 光栅化

绘制调用（draw call）：在计算机图形学中，指向图形处理器发送指令以绘制图形的过程。

光栅化：找出三角形所覆盖到的像素；

## 内容

### [实现加载等待页面Loading Screen设计（进度条异步加载）](https://www.uejoy.com/archives/1071)

启动软件会黑屏半天，或是切换地图黑屏一下，体验不好。

黑屏是因为计算机在启动软件后需要加载磁盘资产，加载过程是需要时间消耗的，但是有时候需要加载的资产又多，然后磁盘读取效率又低，所以导致等待时间过长。在加载的时候，软件窗口是空闲的，但是又没有其他渲染任务，所以就黑屏了。


加载进度条等待页面，有进度条可以直观看到加载进度情况。

整个过程我们需要借助两个函数（全局函数）

* LoadPackageAsync：完成后台的异步加载
* GetAsyncLoadPercentage：获取加载过程中的完成进度（值0-100）


[添加开场mp4视频](https://blog.csdn.net/qq_25580197/article/details/73460060)

[切换虚幻引擎启动时加载的图片Splash](https://blog.csdn.net/qq_31263521/article/details/80567736)

在`项目设置->平台->Windows->Splash->游戏启动画面`中进行图片选择。

Icon默认为：`UnrealEngine\Engine\Build\Windows\Resources\Default.ico`；
新建的Icon放在：`UnrealEngine\Engine\Content\Splash`。

PPT编辑文字，导出PDF，然后裁剪，导出为图片。

图片转icon：https://www.icoconverter.com/

黑白反转：https://invert.imageonline.co/cn/

### 编译
[编译配置参考](https://docs.unrealengine.com/4.26/zh-CN/ProductionPipelines/DevelopmentSetup/BuildConfigurations/)

### 像素流插件
相关源码位于：`unreal\Engine\Plugins\Media\PixelStreaming\PixelStreaming.uplugin`

### 编辑器界面汉化
资源存储位置：`engine\Engine\Content\Localization\Editor\zh-Hans`，只需使用文本编辑器（例如 Notepad++）编辑`Editor.archive`。

### 问题
* 右键`.uproject`文件没有`Switch Unreal Engine version...`
解决：双击`Engine\Binaries\Win64\UnrealVersionSelector-Win64-Shipping.exe`，出现`Register this directory as an Unreal Engine installation?`后点击`是(Y)`。

* 增加`matlab`插件进行虚幻引擎编译，导致启动虚幻编辑器启动失败，原因不明。

* 如果启动虚幻编辑器时出现选择`renderdocui.exe`的选项，则从 [RenderDoc](https://renderdoc.org/) 中下载并安装软件，然后选择相应的exe文件即可启动。 
或者将工程文件 [CarlaUE4.uproject](https://github.com/OpenHUTB/hutb/blob/03c608eb6715267f70270b3412a3aa2ef8d90d57/Unreal/CarlaUE4/CarlaUE4.uproject#L113) 的 `RenderDocPlugin` 配置设置为 `false`，将 RenderDocPlugin 插件禁用。



## [Git 版本管理](https://zhuanlan.zhihu.com/p/104197715)

1. 源码控制->连接到源代码管理->Git；
2. Git路径选择Git安装后bin文件中的启动程序；选择`接收设置`


## 打包报错

* `ERROR: A conflicting instance of UnrealBuildTool is already running.`
> [打开任务结束AutomationTool.exe进程](https://www.cnblogs.com/linqing/p/7107015.html)


## [崩溃报告服务器](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/crash-reporting-in-unreal-engine)

UFS（虚幻文件系统）

[崩溃报告的路径](https://dev.epicgames.com/documentation/en-us/unreal-engine/crash-reporting-in-unreal-engine?application_version=4.27)


## 同步 carla 分支
[SHOWFLAG_ALWAYS_ACCESSIBLE](https://github.com/CarlaUnreal/UnrealEngine/commit/3c2c1f84f16d1474f1aa8eb0ccb651bc7aac231a) ：
定义ShowFlags时有两个宏：SHOWFLAG_ALWAYS_ACCESSIBLE和SHOWFLAG_FIXED_IN_SHIPPING。需要在 Shipping 时使用的应该使用 Always Accessible，不需要在 Shipping 时使用Fixed in Shipping。


## [升级到 4.27-plus]

[Change version 4.27.1](https://github.com/EpicGames/UnrealEngine/commits/4.27-plus/?before=c79de00e3ba31b0b96220f8a0591a93d6770a3e3+665)


## [法律风险](https://www.3dcat.live/share/ue4-vsualization-software/) 

UE4虽然给了完整的源码，但并不意味着全部都能用于发布产品。

[条款](https://www.unrealengine.com/en-US/eula/publishing) 阐述了许可授予的相关问题，其中明确规定了发布的产品中不能包含引擎内容和未经过打包的付费内容。

引擎内容就是UE4编辑器相关的各种工具，小到移动旋转缩放轴，大到刷草刷树的地形编辑工具，这些代码都不能打进自己的项目里。

付费内容是从市场花钱购买的插件或工具，如果未经打包就发布到产品中，就相当于你买了一个模型以后直接发到网上，别人可以免费拷走，侵犯了作者的利益，当然也是不允许的。


## 其他 
- [__虚幻引擎中文文档__](https://bitbucket.org/openhutb/engine_doc/src/master/)

- [虚幻引擎官方中文文档](https://www.unrealengine.com/zh-CN/uses/simulation)

- [代码规范](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/epic-cplusplus-coding-standard-for-unreal-engine)






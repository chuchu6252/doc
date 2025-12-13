# HUTB Docker 开发环境

HUTB 可以完全在 Docker 容器内构建和开发。这种方法消除了对依赖项、Python 版本或系统配置的担忧——一切都已在 Dockerfile 中设置完毕。例如，Dockerfile 支持多个 Python 版本，并预装了所有必需的库。

我们提供两种基于 Docker 的 HUTB 构建方法：**单体模式**和**轻量级模式**。

**单体模式**和**轻量级模式**之间的选择取决于您的预期用途，以及磁盘空间、构建时间等因素，以及您更喜欢完全独立的环境（单体架构）还是重用本地编译的引擎（轻量级架构）。

## 单体模式

此 Docker 镜像旨在与 `docker_tools.py` 脚本一起使用。有关更多信息，请参阅[在 Docker 中构建 Unreal Engine 和 CARLA](build_docker_unreal.md) 。

* 将 [模拟引擎](https://github.com/OpenHUTB/engine) 和 [HUTB](https://github.com/OpenHUTB/hutb) 打包到一个 Docker 镜像中。  
* 需要较长的构建过程，并生成一个较大的最终镜像（通常超过 100 GB）。 
* 提供一个完全独立的环境，所有内容都在 Docker 中编译完成。


## 轻量模式（推荐）

此 Docker 镜像专为开发用途而设计，是构建软件包和使用 HUTB 的推荐方式。与单体模式相比，它提供了更高的灵活性和更快的迭代速度。

* 仅安装构建和运行 HUTB 所需的依赖项（以及 NVIDIA 支持）。
* 需要将主机上现有的 [模拟引擎](https://github.com/OpenHUTB/engine) 构建版本挂载到容器中。
* 构建速度快得多，但依赖于本地编译的模拟引擎文件夹。

---

## 构建 Docker 镜像

### 单体模式

如前所述，单体构建会将 [模拟引擎](https://github.com/OpenHUTB/engine) 和 HUTB 打包到一个 Docker 镜像中。此过程可能需要几个小时，生成的镜像大小可能超过 200 GB。

由于此版本克隆了 Epic Games 私有 GitHub 代码库中的模拟引擎，因此您必须拥有已关联到 GitHub 帐户的有效 Epic 凭据。**如果您尚未进行此设置，请在继续操作之前按照 [此指南](https://www.unrealengine.com/en-US/ue4-on-github) 进行操作**。

要构建单体镜像，请运行以下命令：

```sh
Util/Docker/build.sh --monolith --epic-user <GITHUB_USERNAME> --epic-token <GITHUB_ACCESS_TOKEN>

# 或者为指定 HUTB 分支构建单体镜像
Util/Docker/build.sh --monolith --epic-user <GITHUB_USERNAME> --epic-token <GITHUB_ACCESS_TOKEN> --branch <BRANCH_NAME_OR_TAG>
```

这将创建一个名为 `carla-monolith:<branch>` 的 Docker 镜像

### 轻量模式（开发）

轻量级版本仅安装 HUTB 的依赖项，因此在此阶段无需进行其他设置：

```sh
Util/Docker/build.sh --dev
```
---

## 运行 Docker 镜像

### 单体模式

此镜像不能直接运行，而是通过 docker_tools.py 脚本调用。更多信息请参阅 [在 Docker 中构建 Unreal Engine 和 CARLA](build_docker_unreal.md) 。

```sh
# 构建 HUTB 包
# 注意：该软件包将使用用于构建镜像的分支创建。
./docker_tools.py

# 导入一些资产
./docker_tools.py --input /assets/to/import/path --output /output/path
```

### 轻量模式

轻量模式/开发工作流程要求主机上存在一个符合 HUTB 要求的已编译的模拟引擎文件夹。该文件夹将保留在主机上并挂载到容器中。您可以按照 HUTB 官方文档进行本地构建，也可以使用轻量模式容器在 Docker 中编译：

!!! 注意
    在尝试克隆之前，请确保您的 GitHub 帐户已链接到 Epic 的 UnrealEngine 存储库。

```sh
# Clone a Carla-specific Unreal Engine repository
git clone git@github.com:wambitz/CarlaUnrealEngine.git CarlaUE4

# Export UE4_ROOT so it can be mounted inside the dev container
export UE4_ROOT=$(pwd)/CarlaUE4

# From the CARLA root, start the lightweight container
Util/Docker/run.sh --dev

# Inside the container, build UE4.26 following the official steps:
cd /workspace/unreal-engine
./Setup.sh && ./GenerateProjectFiles.sh && make

# Exit the container
exit
```

构建完模拟引擎后，您可以再次运行轻量模式容器：

!!! 注意
    请确保 `UE4_ROOT` 环境变量指向已编译的模拟引擎构建的根目录。

```sh
Util/Docker/run.sh --dev
```
进入容器后，您可以继续执行正常的 HUTB 构建步骤：

```bash
./Update.sh
make PythonAPI
make CarlaUE4Editor
make package ARGS="--python-version=3.10,3.11,3.12"
```

---

## 使用 Devcontainer 进行 HUTB 服务器/客户端开发

您可以使用轻量模式方案的**Visual Studio Code devcontainer**。此设置会将主机上的目录（包括 UE4）挂载到 Docker 环境中。请注意，单体镜像不太适合 devcontainers，因为它将所有内容都存储在镜像内部。

在您的 HUTB 存储库中创建一个`.devcontainer/devcontainer.json`，包含以下内容的文件：

```json
{
    "name": "CARLA UE4 Dev",
    "image": "carla-development:ue4-20.04",

    "initializeCommand": "./Util/Docker/build.sh --dev --ubuntu-distro 20.04",

    // We do NOT need to set "remoteUser" if the Dockerfile's default user is already correct
    // but you can if you want to be explicit. Also "updateRemoteUserUID" can be false, since
    // our Dockerfile already set the user to our exact UID/GID.
    "updateRemoteUserUID": false,

    "customizations": {
      "vscode": {
        "settings": {
          "terminal.integrated.defaultProfile.linux": "bash"
        },
        "extensions": [
          "ms-vscode.cpptools"
        ]
      }
    },

    // NOTE1: DO NOT pass --user here (we want the Dockerfile default user, not an override)
    // NOTE3: Ensure UE4_ROOT environment variable is defined in host
    "runArgs": [
      "--rm",
      "--runtime", "nvidia",
      "--name", "carla-ue4-development-20.04",
      "--env", "NVIDIA_VISIBLE_DEVICES=all",
      "--env", "NVIDIA_DRIVER_CAPABILITIES=all",
      "--env", "UE4_ROOT=/workspaces/unreal-engine",
      "--env", "CARLA_UE4_ROOT=/workspaces/carla",
      "--env", "DISPLAY=${localEnv:DISPLAY}",
      "--volume", "/tmp/.X11-unix:/tmp/.X11-unix",
      "--volume", "/var/run/docker.sock:/var/run/docker.sock",
      "--volume", "${localEnv:UE4_ROOT}:/workspaces/unreal-engine",
      "--mount", "source=carla-development-ue4-20.04,target=/home/carla",
      "--gpus", "all"
    ]
}
```
---

## 提示与已知问题

1. **在主机上运行二进制文件**  
  **请勿**在容器内构建完成后直接在主机上运行 `make launch`或`make launch-only`，因为容器内部路径（例如 `/workspaces/unreal-engine`）与主机环境不匹配。 如果您需要通过主机访问 HUTB 二进制文件，请先构建一个交付包（`make package` ），然后在主机上从`Dist/`中生成的文件运行它们。

2. **`./Update.sh` 中截断输出**  
   有时，在 HUTB 代码库中运行 `./Update.sh` 可能会截断日志输出。一种解决方法是将输出重定向到一个文件，然后使用 tail 命令查看该文件。

3. **音频支持**  
   ALSA 和 PulseAudio 未预先配置，因此在容器内执行某些脚本时可能会看到音频警告。

---

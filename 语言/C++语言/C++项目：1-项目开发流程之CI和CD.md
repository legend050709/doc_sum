```table-of-contents
```
# 概述
在软件开发和上线部署的流程中，CI/CD 是 DevOps 实践的核心组成部分，它通过自动化流程将代码从开发到交付的各个环节串联起来



# CI 和 CD
CI/CD 实际上代表三个核心概念：持续集成（CI）、持续交付（CD）和持续部署（CD）

**（1）CI（持续集成，Continuous Integration）**：**“频繁合并”**。
开发人员每天多次将代码合并到主干，每次合并都会触发**自动化构建、编译和单元测试**，目的是**尽早发现集成错误**。


**（2）CD（持续交付，Continuous Delivery）**：  **“随时可发”**
是 CI 的进一步延伸。它将通过测试的代码自动部署到测试环境或预生产环境，确保代码随时处于“可发布”的状态。其核心目标是缩短发布准备时间，但最终是否真正发布到生产环境，通常还需要保留人工决策（手动触发发布）

**（3）CD（持续部署，Continuous Deployment）**：  
在持续交付的基础上更进一步。它实现了更高程度的自动化，一旦代码通过了所有必要的自动化测试，就会**自动**部署到生产环境中，全程无需人工手动干预


## CI
**CI（持续集成，Continuous Integration）**：
强调开发过程中的“频繁合并”和“快速验证”。开发人员频繁（例如每天多次）将代码提交到主线代码仓库，如gitlab中；每次提交后系统会自动校验代码能不能正常编译、能不能通过测试（单元测试、集成测试），尽早发现冲突、Bug，确保代码的可集成性和稳定性。

即：**CI 只负责：构建、校验（单元测试）、产出制品（二进制程序/库 / 镜像）；CI【不负责】把程序发布到服务器**。

### 工作时机
**代码提交/合并请求**：开发者把代码提交到 Git（GitLab/GitHub）， 执行了MR的时候。

### CI 做的事情

- 拉取最新代码
- 代码静态检查（lint、规范检测）
- 编译程序
- 运行单元测试、接口测试
- 运行代码扫描（检查有没有安全漏洞）
- 生成镜像 / 程序包（二进制、docker image）


## CD
### CD1：持续交付 Continuous Delivery

**持续交付 = CI 的延伸：CI 产出软件包 / 镜像，制品随时可用，可以一键部署到任意环境。**
重点：**制品已经准备好了，部署动作由人手动触发**。

流程：开发者提交代码 → CI 构建镜像 → 镜像存入镜像仓库；
运维 / 开发**手动点击按钮**，部署到测试环境、预发环境。

特点：
所有版本随时具备部署条件；**人工控制上线时机**。绝大多数互联网公司测试、预发环境使用这套模式。

### CD2：持续部署 Continuous Deployment

**持续部署：完全自动化，CI 构建成功后，自动直接部署到生产环境，无需人工干预。**

> 风险很高，核心业务很少直接生产全自动部署。

流程：代码提交 → CI 构建测试通过 → **自动发布线上**



## CI 和 CD的区别和联系

- **CI：只管「构建 + 验证，产出软件制品」**
- **持续交付 CD：制品就绪，手动一键部署**
- **持续部署 CD：制品就绪，自动上线生产**

|维度|CI（持续集成）|CD（持续交付/部署）|
|---|---|---|
|**核心关注点**|**代码质量**（合代码时别出错）|**发布效率**（代码写完赶紧上线）|
|**触发时机**|**代码提交/合并请求（PR/MR）时**|**代码合并到主干后**（或定时触发）|
|**主要产出**|编译产物（Jar包/Exe文件） + 测试报告|可运行的**线上环境**（生产或预发）|
|**常见操作**|编译、单元测试、静态代码扫描、Lint 检查|构建 Docker 镜像、更新 K8s 配置、数据库迁移|
|**失败后果**|阻塞代码合并（阻止坏代码入库）|阻碍上线（可能导致服务宕机）|
|**执行频率**|**极高**（每次代码提交都跑）|**较高**（每次发版或合并主干跑）|
|**人工介入**|**无**（全自动后台运行）|**可有可无**（持续部署是全自动，持续交付需要点按钮）|

## 完整流程

`开发本地编码 → git push → 触发CI流水线`

1> CI 阶段：代码拉取 → 代码检查 → 编译 → 单元测试 → 构建 Docker 镜像 → 推送镜像仓库
2> 持续交付 CD：**人工触发** → 拉取镜像 → 部署到测试环境
3> 测试验证无误 → **人工触发** → 部署预发
4> 最终人工确认 → 部署生产

国内大厂业务：**CI + 持续交付（最主流）**；极少核心业务使用持续部署。

# 常见的CI/CD工具：GitLab CI
代码托管 + CI/CD 一体化，配置简单（用 `.gitlab-ci.yml` 文件）。
## 基本概念
### Pipeline (流水线)
这是 CI/CD 流程的最高层级。每次你推送代码或创建合并请求，GitLab 就会触发一个 pipeline。它由多个**阶段 (Stages)** 组成，比如 `build`、`test`、`deploy`.

### Stage (阶段)
定义流程的先后顺序（如 `build` -> `test` -> `deploy`）。不同阶段串行执行，前一阶段失败则后续不执行。

### Job (作业)
这是 GitLab CI/CD 的最小执行单元。每个 Job 定义了一系列要执行的命令。同一个 Stage 内的多个 Jobs 可以**并行运行**，这能大大加速你的 CI/CD 流程。

### GitLab CI Runner（执行器）
Runner 是真正“跑”这些 Job 的“工人”/代理程序。它是一个独立的应用程序（独立于Gitlab服务），负责从 GitLab 接收 Job，在指定的环境（比如一个干净的 Docker 容器）中执行脚本，然后将结果（成功/失败、日志、产物）报告给 GitLab。
GitLab 提供官方托管的 Runner，你也可以注册自己的 Runner 以获得更高的定制性和安全性。

- 支持多种执行器：Docker（最常用）、Shell、K8s、虚拟机；
- 分为共享 Runner（GitLab 公共）、私有 Runner（公司内部自建，生产推荐）；
- 收到 GitLab 调度任务后，拉取镜像、拉取代码、执行脚本、上传产物。

### 其他
#### Artifacts（构建产物）
用于在不同阶段之间传递文件（如编译好的二进制文件或测试报告）。

#### Cache（缓存）
用于存储依赖项（如第三方库），加速后续构建。

#### Rules（规则）
控制 Job 在何种条件（如特定分支、标签）下触发。


## 标准执行流程

**（1）触发**：开发者执行`git push`/ 创建 MR，GitLab 检测到代码变更。
    
**（2）创建**：GitLab 根据你项目根目录下的 `.gitlab-ci.yml` 文件，创建一个新的 Pipeline。
    
**（3）排队**：Pipeline 中的 Job 被放入队列，等待可用的 Runner。
    
**（4）获取**：一个匹配的空闲  Runner（例如，拥有特定 tag）从 GitLab 获取一个 Job。
    
**（5）执行**：Runner 准备好执行环境（比如：隔离环境，如 Docker 容器），然后按顺序执行你在 `.gitlab-ci.yml` 中为该 Job 定义的 `script` 命令。
注： Job 执行失败则阻断后续 Stage；执行成功可产出产物（二进制、镜像）；
    
**（6）报告**：所有 Stage 执行完成，流水线结束；Runner 将执行日志和最终状态（成功/失败）实时传回 GitLab，你可以在 GitLab 的 Web 界面上查看。


## gitlab-ci.yml

所有流程都通过项目根目录下的 `.gitlab-ci.yml` 文件定义。它是一个 YAML 文件，描述了整个 Pipeline 的结构，GitLab 识别该文件后自动启用流水线。

### 关键字

#### 顶层关键字

|关键字|作用|
|---|---|
|`stages`|定义流水线执行阶段，顺序从上到下；不定义默认 build/test/deploy|
|`variables`|全局环境变量，所有 Job 共享；敏感变量在 GitLab 后台配置|
|`cache`|缓存三方库、编译中间文件，大幅减少重复下载编译耗时|

#### Job 内核心关键字

|关键字|作用|
|---|---|
|`stage`|指定当前 Job 归属哪个阶段|
|`image`|指定 Runner 使用的 Docker 基础镜像（C++ 用 gcc 镜像）|
|`script`|核心执行脚本，数组格式，一条条顺序执行|
|`before_script` / `after_script`| 在 `script` 之前/之后执行的命令，常用于安装依赖或清理环境|
|`artifacts`|Job 产出文件，保存至 GitLab，下游 Job 可依赖获取。在 Job 执行成功后，将指定的文件或目录（如编译好的二进制文件、测试报告）传递给后续的 Job，或允许用户下载|
|`dependencies`|指定依赖上游 Job，自动拉取上游 artifacts 产物|
|`only / rules`|用于控制 Job 在什么条件下才被触发执行。例如，只有 `main` 分支的推送才触发部署 Job，或者只有打了 Git Tag 才发布新版本|
|`when: manual`|手动触发 Job，不会自动执行，上线部署专用|
|`services`|配套服务容器，构建 Docker 镜像必须`docker:dind`|

## 范例：完整标准流程（C++ RPC 项目）
```bash
开发者本地编码 → git push / 提交MergeRequest
        ↓
GitLab 读取 .gitlab-ci.yml 生成Pipeline
        ↓
Stage1 build（编译C++ RPC程序）
        ↓
Stage2 test（单元测试ctest，失败直接终止流水线）
        ↓
Stage3 package（构建Docker镜像，推送镜像仓库）
        ↓
Stage4 deploy（手动触发，部署到测试/预发/生产服务器/K8s）
```
### 目录结构
```bash
rpc-project/
├── .gitlab-ci.yml    # CI流水线配置
├── Dockerfile         # 多阶段构建镜像
├── CMakeLists.txt     # C++编译配置
├── src/                # brpc rpc源码
├── conf/               # 服务配置文件
└── thirdparty/         # 三方依赖
```

###  gitlab-ci.yml 编写

```yaml
# 使用一个包含 gRPC 和 C++ 编译工具的 Docker 镜像作为基础运行环境
image: gcc:latest

# 定义 Pipeline 的阶段。它们将按此顺序执行
stages:
  - build          # 编译阶段
  - test           # 测试阶段
  # - deploy       # 部署阶段 (本例中略去)

# 全局缓存，这里用于缓存 CMake 的构建目录，加速后续构建
cache:
  paths:
    - build/

# ---------- Build 阶段 Job ----------
# 负责编译你的 RPC 库和客户端/服务器端示例
build-job:
  stage: build
  script:
    - echo "Starting build for gRPC C++ RPC project..."
    # 1. 创建并进入构建目录
    - mkdir -p build
    - cd build
    # 2. 使用 CMake 配置项目。这里假设你已正确安装 gRPC 和 Protobuf
    - cmake .. -DCMAKE_BUILD_TYPE=Release
    # 3. 开始编译
    - make -j$(nproc)
  artifacts:
    # 将编译好的可执行文件作为产物，传递给后续的 test 阶段
    paths:
      - build/greeter_server
      - build/greeter_client

# ---------- Test 阶段 Job ----------
# 依赖 build-job 产出的可执行文件，并启动服务器和客户端进行测试
test-job:
  stage: test
  needs:
    - build-job   # 明确声明此 Job 依赖于 build-job 的成功完成和产物
  script:
    - echo "Starting test for gRPC C++ RPC project..."
    # 1. 进入构建目录
    - cd build
    # 2. 在后台启动 RPC 服务器
    - ./greeter_server &
    # 3. 等待服务器完全启动（生产环境推荐更健壮的检查）
    - sleep 2
    # 4. 运行 gRPC 客户端，它会向服务器发送请求并验证响应
    - ./greeter_client
    # 如果客户端命令执行失败（返回非零值），此 Job 将失败，Pipeline 也会停止
```

### 进阶优化
在实际的企业级 C++ 项目中，还可以引入以下优化：
**（1）静态代码分析**：在 `build` 和 `test` 之间增加 `analyze` 阶段，集成 `cppcheck` 或 `clang-tidy`，在编译阶段就拦截不安全的代码（如内存泄漏风险）。

**（2）编译加速**：配置 `ccache` 缓存已编译的目标文件，或者使用分布式编译工具 `distcc`，大幅缩短 C++ 项目的编译时间。

**（3）环境隔离**：强烈建议使用 Docker Executor 或 Kubernetes Executor 运行 C++ 构建任务，避免不同项目间的系统依赖冲突

# 参考
```bash

```
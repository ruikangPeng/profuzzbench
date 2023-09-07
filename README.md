# ProFuzzBench：一个用于状态协议模糊化的基准
ProFuzzBench 是网络协议状态模糊化的基准。它包括一套用于流行协议（如TLS、SSH、SMTP、FTP、SIP）的代表性开源网络服务器，以及自动化实验的工具。

# 引用 ProFuzzBench

ProFuzzBench 已被接受作为[工具演示论文](https://dl.acm.org/doi/pdf/10.1145/3460319.3469077)在 2021 年第 30 届 ACM SIGSOFT 软件测试与分析国际研讨会(ISSTA)上发表。

```
@inproceedings{profuzzbench,
  title={ProFuzzBench: A Benchmark for Stateful Protocol Fuzzing},
  author={Roberto Natella and Van-Thuan Pham},
  booktitle={Proceedings of the 30th ACM SIGSOFT International Symposium on Software Testing and Analysis},
  year={2021}
}
```

# 文件夹结构
```
protocol-fuzzing-benchmark
├── subjects: 该文件夹包含该基准测试中包含的所有协议，并且每个协议可能有多个目标服务器
│   └── RTSP
│   └── FTP
│   │   └── LightFTP
│   │       └── Dockerfile: Dockerfile
│   │       └── run.sh: 在容器内运行实验的主脚本
│   │       └── cov_script.sh: 执行代码覆盖率分析的脚本
│   │       └── other files (例如，补丁、其他的脚本)
│   └── ...
└── scripts: 该文件夹包含运行实验、收集和分析结果的所有脚本
│   └── execution
│   │   └── profuzzbench_exec_common.sh: 生成容器并在容器上运行实验的主脚本
│   │   └── ...
│   └── analysis
│       └── profuzzbench_generate_csv.sh: 该脚本收集不同运行的代码覆盖率结果
│       └── profuzzbench_plot.py: 打印示例脚本
└── README.md
```


# Fuzzers

ProFuzzBench 提供了三个模糊器来模糊目标的自动化脚本：[AFLnwe](https://github.com/aflnet/aflnwe)(AFL的一个网络启用版本，它通过TCP/IP套接字而不是文件发送输入)、[AFLNet](https://github.com/aflnet/aflnet)(一个为有状态网络服务器量身定制的模糊器)和 [StateAFL](https://github.com/stateafl/stateafl)(另一个用于有状态网络服务器的模糊器)。

在下面的教程中，您可以找到运行 AFLnwe 和 AFLnet(ProFuzzBench支持的前两个模糊器)的说明。有关 StateAFL 的更多信息，请查看 [README-StateAFL.md](README-StateAFL.md)。


# 教程-用 [AFLNet](https://github.com/aflnet/aflnet)和 [AFLnwe](https://github.com/aflnet/aflnwe)模糊 LightFTP 服务器，AFL 的网络启用版本
按照以下步骤运行并收集 LightFTP 的实验结果，LightFTP 是一种轻量级文件传输协议(FTP)服务器。在其他受试者身上进行实验应该遵循类似的步骤。每个受试者程序都附带一个 README.md 文件，其中显示了运行实验的受试者特定命令。

## Step-0. 设置环境变量
```
git clone https://github.com/profuzzbench/profuzzbench.git
cd profuzzbench
export PFBENCH=$(pwd)
export PATH=$PATH:$PFBENCH/scripts/execution:$PFBENCH/scripts/analysis
```

## Step-1. 构建 docker 镜像
以下命令创建一个标记为 lightftp 的 docker 镜像。镜像应该具有可用于模糊处理和代码覆盖率收集的所有内容。

```bash
cd $PFBENCH
cd subjects/FTP/LightFTP
docker build . -t lightftp
```

## Step-2. Run fuzzing
运行 [profuzzbench_exec_common.sh script](scripts/execution/profuzzbench_exec_common.sh)脚本开始一个实验。该脚本包含 8 个参数，如下所示。

- ***1st argument (DOCIMAGE)*** : docker 镜像的名称
- ***2nd argument (RUNS)***     : 运行次数，每次运行生成一个独立的 Docker 容器
- ***3rd argument (SAVETO)***   : 保存结果的文件夹的路径
- ***4th argument (FUZZER)***   : fuzzer 名称(例如，aflnet)--此名称必须与 Docker 容器内的 fuzzer 文件夹的名称匹配(例如/home/ubuntu/aflnet)
- ***5th argument (OUTDIR)***   : 在 docker 容器内创建的输出文件夹的名称
- ***6th argument (OPTIONS)***  : 除了在特定于目标的 run.sh 脚本中编写的标准选项之外，fuzzing 所需的所有选项
- ***7th argument (TIMEOUT)***  : 模糊时间(秒)
- ***8th argument (SKIPCOUNT)***: 用于计算随时间变化的覆盖范围。例如，SKIPCOUNT=5 意味着我们在每 5 个测试用例之后运行 gcovr，因为 gcovr 需要时间，并且我们不希望在每个测试用例之后都运行它

以下命令运行 4 个 AFLNet 实例和 4 个 AFLnwe 实例，在 60 分钟内同时模糊 LightFTP。

```bash
cd $PFBENCH
mkdir results-lightftp

profuzzbench_exec_common.sh lightftp 4 results-lightftp aflnet out-lightftp-aflnet "-P FTP -D 10000 -q 3 -s 3 -E -K" 3600 5 &
profuzzbench_exec_common.sh lightftp 4 results-lightftp aflnwe out-lightftp-aflnwe "-D 10000 -K" 3600 5
```

如果脚本成功运行，其输出应类似于下面的文本。

```
AFLNET: Fuzzing in progress ...
AFLNET: Waiting for the following containers to stop:  f2da4b72b002 b7421386b288 cebbbc741f93 5c54104ddb86
AFLNET: Collecting results and save them to results-lightftp
AFLNET: Collecting results from container f2da4b72b002
AFLNET: Collecting results from container b7421386b288
AFLNET: Collecting results from container cebbbc741f93
AFLNET: Collecting results from container 5c54104ddb86
AFLNET: I am done!
```

## Step-3. 收集结果
所有结果(在tar文件中)都应存储在步骤 2 中创建的文件夹中(results-lightftp)。具体来说，这些 tar 文件是所有模糊实例生成的输出文件夹的压缩版本。如果模糊器是基于 afl 的(例如，AFLNet、AFLnwe)，则每个文件夹都应该包含子文件夹，如崩溃、挂起、队列等。使用 [profuzzbench_generate_csv.sh script](scripts/analysis/profuzzbench_generate_csv.sh)脚本收集随时间变化的代码覆盖率结果。该脚本包含 5 个参数，如下所示。

- ***1st argument (PROG)***   : 待测程序的名称(例如，lightftp)
- ***2nd argument (RUNS)***   : 运行次数
- ***3rd argument (FUZZER)*** : 模糊器名称(例如aflnet)
- ***4th argument (COVFILE)***: 保存结果的 CSV 格式输出文件
- ***5th argument (APPEND)*** : 附加模式；对于第一个模糊器，将其设置为 0，对于后续模糊器，设置为 1。

以下命令收集由 AFLNet 和 AFLnwe 生成的代码覆盖率结果，并将其保存到 results.csv。

```bash
cd $PFBENCH/results-lightftp

profuzzbench_generate_csv.sh lightftp 4 aflnet results.csv 0
profuzzbench_generate_csv.sh lightftp 4 aflnwe results.csv 1
```

results.csv 文件应与下面的文本类似。该文件有六列，显示时间戳、主题程序、模糊器名称、运行索引、覆盖率类型及其值。该文件包含线路覆盖率和分支覆盖率随时间变化的信息。每个覆盖率类型都有两个值，分别为百分比 percentage(*_per) 和绝对数 absolute number(*_abs)。


```
time,subject,fuzzer,run,cov_type,cov
1600905795,lightftp,aflnwe,1,l_per,25.9
1600905795,lightftp,aflnwe,1,l_abs,292
1600905795,lightftp,aflnwe,1,b_per,13.3
1600905795,lightftp,aflnwe,1,b_abs,108
1600905795,lightftp,aflnwe,1,l_per,25.9
1600905795,lightftp,aflnwe,1,l_abs,292
```

## Step-4. 分析结果
步骤 3 中收集的结果(即result.csv)可用于绘图。例如，我们提供了一个[示例 Python 脚本](scripts/analysis/profuzzbench_plot.py)来绘制随时间变化的代码覆盖率。使用以下命令打印结果并将其保存到文件中。

```
cd $PFBENCH/results-lightftp

profuzzbench_plot.py -i results.csv -p lightftp -r 4 -c 60 -s 1 -o cov_over_time.png
```

这是由脚本生成的示例代码覆盖率报告。
![Sample report](figures/cov_over_time.png)

# 实用程序脚本

ProFuzzBench 还包括用于在所有 targe 上运行所有模糊器的脚本，并带有预先配置的参数。要为所有模糊器构建所有目标，可以运行脚本 [profuzzbench_build_all.sh](scripts/execution/profuzzbench_build_all.sh)。要运行模糊器，可以使用脚本 [profuzzbench_exec_all.sh](scripts/execution/profuzzbench_exec_all.sh)。


# 并行生成

为了加快构建 Docker 镜像的速度，你可以通过传入"-j"去`make`，使用 `MAKE_OPT`环境变量和 `docker build` 的 `--build-arg`选项。示例：

```
export MAKE_OPT="-j4"
docker build . -t lightftp --build-arg MAKE_OPT
```

# FAQs

## 1. 如何扩展 ProFuzzBench？

如果您想添加新协议或新目标服务器(支持的协议)，请按照上述文件夹结构并完成以下步骤。我们以 LightFTP 为例。

### Step-1. 创建包含协议/目标服务器的新文件夹

LightFTP 服务器的文件夹为 [subjects/FTP/LightFTP](subjects/FTP/LightFTP)。

### Step-2. 为新的目标服务器编写 Docker 文件，并准备所有特定于主题的脚本/文件(例如，特定于目标的补丁、种子语料库)

下面的文件夹结构显示了我们为模糊 LightFTP 服务器准备的所有文件。请阅读我们的论文以了解这些文件的用途。

```
subjects/FTP/LightFTP
├── Dockerfile (required): 在此基础上，构建了一个特定于目标的 Docker 镜像(请参阅教程中的步骤1)
├── run.sh (required): 在容器中运行实验的主脚本
├── cov_script.sh (required): 执行代码覆盖率分析的脚本
├── clean.sh (optional): 在模糊处理之前清理服务器状态以提高稳定性的脚本
├── fuzzing.patch (optional): 改进模糊结果所需的代码更改(例如，消除随机性)
├── gcov.patch (required): 支持代码覆盖率分析所需的代码更改(例如，启用 gcov，添加信号处理程序)
├── ftp.dict (optional): 一个字典，包含支持模糊处理的协议特定 tokens/关键字
└── in-ftp (required): 种子语料库，捕获发送到被测试服务器的客户端请求的序列。
│   │       要准备这些种子，请按照 AFLNet 教程 https://github.com/aflnet/aflnet.
│   │       请对所有种子输入使用".raw"扩展名。
│   │
│   └── ftp_requests_full_anonymous.raw
│   └── ftp_requests_full_normal.raw
└── README.md (optional): 特定于目标的自述文件，包含运行实验的命令
```
所有必需的文件(即 Dockerfile、run.sh、cov_script.sh、gcov.patch 和种子语料库)都遵循一些模板，这样就可以很容易地按照它们为新目标准备文件。

### Step-3. 测试您的新目标服务器

一旦成功构建了 Docker 镜像，就应该在单个 Docker 容器中测试在特定于目标的 [README.md](subjects/FTP/LightFTP/README.md)中编写的命令。例如，我们运行以下命令来检查 LightFTP 是否一切正常。

```
//启动容器
docker run -it lightftp /bin/bash

//在docker 容器中
//使用 AFLNet 进行 60 分钟的模糊实验
cd experiments
run aflnet out-lightftp-aflnet "-P FTP -D 10000 -q 3 -s 3 -E -K -c ./ftpclean.sh" 3600 5
```

如果一切正常，应该没有错误消息，所有结果都存储在 out-lightftp-aflnet 文件夹中。

## 2. 我的实验 "hangs"。原因可能是什么？

每个实验有两个部分：模糊和代码覆盖率分析。模糊部分应在指定的超时后完成；然而，代码覆盖率分析时间是特定于被测对象的，如果生成的语料库很大或目标服务器速度较慢，则可能需要几个小时。如果你认为你的实验挂起了，你可以登录到正在运行的容器来检查进度。


## 3. 如何将另一个模糊器添加到 ProFuzzBench？

为了增加对额外模糊器的支持，我们建议在目标的文件夹中添加一个新的 Dockerfile，并在目标的镜像上构建模糊器。 例如，StateAFL 模糊器已添加为 ProFuzzBench 的扩展。对于每个支持的目标，您将找到文件 `Dockerfile-stateafl`，以便在其他模糊器的同一容器镜像中构建模糊器。在 Dockerfile中，您可以添加特定于模糊器的指令。例如，StateAFL 还重新构建了添加更多编译时指令插入的目标。

此外，您可以在目标文件夹中包含 `run-${FUZZER}.sh` 脚本，例如 `run-stateafl.sh`。当运行模糊实验时，该文件将由 `run.sh`脚本提供。在这个脚本中，您可以包含特定于模糊器的命令，例如设置环境变量(例如，目标的插入指令的构建的文件夹)。
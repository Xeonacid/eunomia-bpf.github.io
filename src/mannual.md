# eunomia-bpf 用户手册: 让 eBPF 程序的开发和部署尽可能简单

<!-- TOC -->

- [从 _C 语言_ 的 Hello World 开始](#从-c-语言-的-hello-world-开始)
- [eunomia-bpf 的 Hello World](#eunomia-bpf-的-hello-world)
- [添加 map 记录数据](#添加-map-记录数据)
- [使用 ring buffer 往用户态发送数据](#使用-ring-buffer-往用户态发送数据)
- [使用 perf event array 往用户态发送数据](#使用-perf-event-array-往用户态发送数据)
- [使用 github-template 实现远程编译](#使用-github-template-实现远程编译)
- [通过 API 进行热插拔和分发](#通过-api-进行热插拔和分发)
- [使用 Prometheus 或 OpenTelemetry 进行可观测性数据收集](#使用-prometheus-或-opentelemetry-进行可观测性数据收集)
  - [example](#example)
- [使用说明](#使用说明)
- [原理](#原理)
- [为我们的项目贡献代码](#为我们的项目贡献代码)

<!-- /TOC -->

传统来说， eBPF 的开发方式主要有 BCC、libbpf 等方式。要完成一个 BPF 二进制程序的开发，需要搭建开发编译环境，要关注目标系统的内核版本情况，需要掌握从 BPF 内核态到用户态程序的编写，以及如何加载、绑定至对应的 HOOK 点等待事件触发，最后再对输出的日志及数据进行处理。

我们希望有这样一种 eBPF 的编译和运行工具链，就像其他很多语言一样：

- 大多数用户只需要关注 `bpf.c` 程序本身的编写，不需要写任何其他的什么 Python, Clang 之类的用户态辅助代码框架；
  这样我们可以很方便地分发、重用 eBPF 程序本身，而不需要和某种或几种语言的生态绑定；

- 最大程度上和主流的 libbpf 框架实现兼容，原先使用 libbpf 框架编写的代码几乎不需要改动即可移植；eunomia-bpf 编写的 eBPF 程序也可以使用 libbpf 框架来直接编译运行；

- 本地只需要下载一个很小的二进制运行时，没有任何的 Clang LLVM 之类的大型依赖，可以支持热插拔、热更新；
  也可以作为 Lua 虚拟机那样的小模块直接编译嵌入其他的大型软件中，提供 eBPF 程序本身的服务；运行和启动时资源占用率都很低；

- 让 eBPF 程序的分发和使用像网页和 Web 服务一样自然（Make eBPF as a service）：
  支持在集群环境中直接通过一次请求进行分发和热更新，仅需数十 kB 的 payload，
  <100ms 的更新时间，和少量的 CPU 内存占用即可完成 eBPF 程序的分发、部署和更新；
  不需要执行额外的编译过程，就能得到 CO-RE 的运行效率；

## 从 _C 语言_ 的 Hello World 开始

还记得您第一次写 _C 语言_ 的 **Hello World 程序** 吗？首先，我们需要一个 `.c` 文件，它包含一个 `main` 函数：

```c
int main(void)
{
    printf("Hello, World!\n");
    return 0;
}
```

我们叫它 `hello.c`，接下来就只需要这几个步骤就好：

```bash
# if you are using Ubuntu without a c compiler
sudo apt insalll build-essentials
# compile the program
gcc -o hello hello.c
# run the program
./hello
```

只需要写一个 c 文件，执行两行命令就可以运行；大多数情况下你也可以把编译好的可执行文件直接移动到其他同样架构的机器或不同版本的操作系统上，然后运行它，也会得到一样的结果:

```plaintext
Hello World!
```

## eunomia-bpf 的 Hello World

首先，我们需要一个 `bpf.c` 文件，它就是正常的、合法的 C 语言代码，和 libbpf 所使用的完全相同：

```c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>

typedef int pid_t;

char LICENSE[] SEC("license") = "Dual BSD/GPL";

SEC("tp/syscalls/sys_enter_write")
int handle_tp(void *ctx)
{
 pid_t pid = bpf_get_current_pid_tgid() >> 32;
 bpf_printk("BPF triggered from PID %d.\n", pid);
 return 0;
}
```

假设它叫 `hello.bpf.c`，新建一个 `/path/to/repo` 的文件夹并且把它放进去，接下来的步骤：

```console
# 下载安装 ecli 二进制
wget https://aka.pw/bpf-ecli -O /usr/local/ecli && chmod +x /usr/local/ecli
# 使用容器进行编译，生成一个 package.json 文件，里面是已经编译好的代码和一些辅助信息
docker run -it -v /path/to/repo:/src yunwei37/ebpm:latest
# 运行 eBPF 程序（root shell）
sudo ecli run package.json
```

> 使用 docker 的时候需要把包含 .bpf.c 文件的目录挂载到容器的 /src 目录下，目录中只有一个 .bpf.c 文件；

它会追踪所有进行 write 系统调用的进程的 pid：

```console
$ sudo cat /sys/kernel/debug/tracing/trace_pipe
cat-42755   [003] d...1 48755.529860: bpf_trace_printk: BPF triggered from PID 42755.
             cat-42755   [003] d...1 48755.529874: bpf_trace_printk: BPF triggered from PID 42755.
```

我们编译好的 eBPF 代码同样可以适配多种内核版本，可以直接把 package.json 复制到另外一个机器上，然后不需要重新编译就可以直接运行（CO-RE：Compile Once Run Every Where）；也可以通过网络传输和分发 package.json，通常情况下，压缩后的版本只有几 kb 到几十 kb。

## 添加 map 记录数据

参考：<https://github.com/eunomia-bpf/eunomia-bpf/tree/master/examples/bpftools/bootstrap>

```c

struct {
 __uint(type, BPF_MAP_TYPE_HASH);
 __uint(max_entries, 8192);
 __type(key, pid_t);
 __type(value, u64);
} exec_start SEC(".maps");

```

添加 map 的功能和 libbpf 没有任何区别，只需要在 .bpf.c 中定义即可。

## 使用 ring buffer 往用户态发送数据

参考：<https://github.com/eunomia-bpf/eunomia-bpf/tree/master/examples/bpftools/bootstrap>

只需要定义一个头文件，包含你想要发送给用户态的数据格式，以 `.bpf.h` 作为后缀名：

```c
/* SPDX-License-Identifier: (LGPL-2.1 OR BSD-2-Clause) */
/* Copyright (c) 2020 Facebook */
#ifndef __BOOTSTRAP_H
#define __BOOTSTRAP_H

#define TASK_COMM_LEN 16
#define MAX_FILENAME_LEN 127

struct event {
 int pid;
 int ppid;
 unsigned exit_code;
 unsigned long long duration_ns;
 char comm[TASK_COMM_LEN];
 char filename[MAX_FILENAME_LEN];
 unsigned char exit_event;
};

#endif /* __BOOTSTRAP_H */
```

在代码中定义环形缓冲区之后，就可以直接使用它：

```c
struct {
 __uint(type, BPF_MAP_TYPE_RINGBUF);
 __uint(max_entries, 256 * 1024);
} rb SEC(".maps");

SEC("tp/sched/sched_process_exec")
int handle_exec(struct trace_event_raw_sched_process_exec *ctx)
{
    ......
 e->exit_event = false;
 e->pid = pid;
 e->ppid = BPF_CORE_READ(task, real_parent, tgid);
 bpf_get_current_comm(&e->comm, sizeof(e->comm));
 /* successfully submit it to user-space for post-processing */
 bpf_ringbuf_submit(e, 0);
 return 0;
}
```

eunomia-bpf 会自动去源代码中找到对应的 ring buffer map，并且把 ring buffer 和类型信息记录在编译好的信息中，并在运行的时候自动完成对于 ring buffer 的加载、导出事件等工作。所有的 eBPF 代码和原生的 libbpf 程序没有任何区别，使用 eunomia-bpf 开发的代码也可以在 libbpf 中无需任何改动即可编译运行。

## 使用 perf event array 往用户态发送数据

使用 perf event 的原理和使用 ring buffer 非常类似，使用我们的框架时，也只需要在头文件中定义好所需导出的事件，然后定义一下 perf event map：

```c
struct {
 __uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
 __uint(key_size, sizeof(u32));
 __uint(value_size, sizeof(u32));
} events SEC(".maps");
```

可以参考：<https://github.com/eunomia-bpf/eunomia-bpf/tree/master/examples/bpftools/opensnoop> 它是直接从 libbpf-tools 中移植的实现；

## 使用 github-template 实现远程编译

由于 eunomia-bpf 的编译和运行阶段完全分离，可以实现在 github 网页上编辑之后，通过 github actions 来完成编译，之后在本地一行命令即可启动：

1. 将此 [github.com/eunomia-bpf/ebpm-template](https://github.com/eunomia-bpf/ebpm-template) 用作 github 模板：请参阅 [creating-a-repository-from-a-template](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-repository-from-a-template)
2. 修改 bootstrap.bpf.c， commit 并等待工作流停止
3. 我们配置了 github pages 来完成编译好的 json 的导出，之后就可以实现 ecli 使用远程 url 一行命令即可运行：

```console
sudo ./ecli run https://eunomia-bpf.github.io/ebpm-template/package.json
```

## 通过 API 进行热插拔和分发

由于 eunomia-cc 编译出来的 ebpf 程序代码和附加信息很小（约数十 kb），且不需要同时传递任何的额外依赖，因此我们可以非常方便地通过网络 API 直接进行分发，也可以在很短的时间（大约 100ms）内实现热插拔和热更新。我们提供了一个简单的 client 和 server，请参考;

[https://github.com/eunomia-bpf/eunomia-bpf/blob/master/documents/ecli-usage.md](https://github.com/eunomia-bpf/eunomia-bpf/blob/master/documents/ecli-usage.md)

之前也有一篇比赛项目的可行性验证的文章：

[https://zhuanlan.zhihu.com/p/555362934](https://zhuanlan.zhihu.com/p/555362934)

## 使用 Prometheus 或 OpenTelemetry 进行可观测性数据收集

基于 async Rust 的 Prometheus 或 OpenTelemetry 自定义可观测性数据收集器: [eunomia-exporter](https://github.com/eunomia-bpf/eunomia-bpf/tree/master/eunomia-exporter)

可以自行编译或通过 [release](https://github.com/eunomia-bpf/eunomia-bpf/releases/) 下载

#### example

这是一个 `opensnoop` 程序，追踪所有的打开文件，源代码来自 [bcc/libbpf-tools](https://github.com/iovisor/bcc/blob/master/libbpf-tools/opensnoop.bpf.c), 我们修改过后的源代码在这里: [examples/bpftools/opensnoop](https://github.com/eunomia-bpf/eunomia-bpf/tree/master/examples/bpftools/opensnoop)

在编译之后，可以定义一个这样的配置文件:

```yml
programs:
  - name: opensnoop
    metrics:
      counters:
        - name: eunomia_file_open_counter
          description: test
          labels:
            - name: pid
            - name: comm
            - name: filename
              from: fname
    compiled_ebpf_filename: examples/bpftools/opensnoop/package.json
```

然后，您可以在任何地方使用 `config.yaml` 和预编译的 eBPF 数据 `package.json` 启动 Prometheus 导出器，您可以看到如下指标：

![prometheus](https://oss.openanolis.cn/sig/stxfomyiiwdwkdrqwlnn)

您可以在任何内核版本上部署导出器，而无需依赖 `LLVM/Clang`。 有关详细信息，请参阅 [eunomia-exporter](https://github.com/eunomia-bpf/eunomia-bpf/tree/master/eunomia-exporter)。

## 使用说明

1. 我们使用与 libbpf 相同的 c eBPF 代码，因此大多数 libbpf eBPF c 代码无需任何修改即可运行。
2. 支持的 eBPF 程序类型：`kprobe`, `tracepoint`, `fentry`, `lsm` 未来我们会增加更多；
3. 如果你想使用环形缓冲区来导出事件，你需要添加 `your_program.bpf.h` 到你的 repo，并在其中定义导出数据类型，导出数据类型应该是 C struct，例如：

   ```c
   struct process_event {
       int pid;
       int ppid;
       unsigned exit_code;
       unsigned long long duration_ns;
       char comm[TASK_COMM_LEN];
       char filename[MAX_FILENAME_LEN];
       int exit_event;
   };
   ```

   导出的名称和字段类型不受限制，但最好使用标准 C 类型。如果标头中存在多个 struct，我们现阶段将使用第一个作为导出的数据类型。仅当我们发现 eBPF 程序中存在 `BPF_MAP_TYPE_RINGBUF` 的 map 时，才启用环形缓冲区导出的功能。

4. 目前部分低版本内核对于 eBPF 模块的支持并不完善，同时也可能没有附带 BTF 信息，因此有可能一些 eBPF 程序在低版本内核上运行时不能得到预期的结果。详情请参考 libbpf 对应的文档：[libbpf/libbpf](https://github.com/libbpf/libbpf) 我们未来会在这方面继续提升兼容性。

## 原理

```mermaid
graph TD
  b3-->package
  b4-->a3
  package-->a1
  package(可通过网络或其他任意方式进行分发: CO-RE)

  subgraph 运行时加载器库
  a3(运行 WASM 模块配置 eBPF 程序或和 eBPF 程序交互)
  a1(根据 JSON 配置信息动态装载 eBPF 程序)
  a2(根据类型信息和内存布局信息对内核态导出事件进行动态处理)
  a1-->a2
  a3-->a1
  a2-->a3
  end

  subgraph eBPF编译工具链
  b1(使用 Clang 编译 eBPF 程序获得包含重定位信息的 bpf.o)
  b2(添加从 eBPF 源代码获取的内核态导出数据的内存布局, 类型信息等)
  b3(打包生成 JSON 数据)
  b4(打包成 WASM 模块进行分发)
  b5(可选的用户态数据处理程序编译为 WASM)
  b2-->b3
  b3-->b5
  b5-->b4
  b1-->b2
  end
```

## 为我们的项目贡献代码

我们的项目还在早期阶段，因此非常希望有您的帮助：

- 运行时库地址： [https://github.com/eunomia-bpf/eunomia-bpf](https://github.com/eunomia-bpf/eunomia-bpf)
- 编译器地址： [https://github.com/eunomia-bpf/eunomia-cc](https://github.com/eunomia-bpf/eunomia-cc)
- 文档：[https://github.com/eunomia-bpf/eunomia-bpf.github.io](https://github.com/eunomia-bpf/eunomia-bpf.github.io)

eunomia-bpf 也已经加入了龙蜥社区：

- gitee 镜像：<https://gitee.com/anolis/eunomia>

您可以帮助我们添加测试或者示例，可以参考：

- [https://github.com/eunomia-bpf/eunomia-bpf/tree/master/examples/bpftools](https://github.com/eunomia-bpf/eunomia-bpf/tree/master/examples/bpftools)
- [https://github.com/eunomia-bpf/eunomia-bpf/tree/master/bpftools/tests](https://github.com/eunomia-bpf/eunomia-bpf/tree/master/bpftools/tests)

由于现在 API 还不稳定，如果您在试用中遇到任何问题或者任何流程/文档不完善的地方，请在 gitee 或 github issue 留言，
我们会尽快修复；也非常欢迎进一步的 PR 提交和贡献！也非常希望您能提出一些宝贵的意见或者建议！

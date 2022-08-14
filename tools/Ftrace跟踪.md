## Ftrace 跟踪技术
1. /sys/kernel/debug/tracing 目录文件说明
是否开启ftrace能力，可以查看`kernel.ftrace_enabled = 1`是否开启
```
$ sysctl -a | grep ftrace
kernel.ftrace_enabled = 1
```

| 文件     | 描述 |
| ----------- | ----------- |
| available_tracers      | 可用跟踪器，hwlat blk function_graph wakeup_dl wakeup_rt wakeup function nop，nop 表示不使用跟踪器       |
| current_tracer   | 当前使用的跟踪器     |
| function_profile_enabled      | 启用函数性能分析器       |
| available_filter_functions   | 可跟踪的完整函数列表    |
| set_ftrace_filter      | 选择跟踪函数的列表，支持批量设置，例如 *tcp、tcp* 和 *tcp* 等       |
| set_ftrace_notrace   | 设置不跟踪的函数列表        |
| set_event_pid      | 设置跟踪的 PID，表示仅跟踪 PID 程序的函数或者其他跟踪       |
| tracing_on   | 是否启用跟踪，1 启用跟踪 0 关闭跟踪        |
| trace_stat（目录）      | 函数性能分析的输出目录   |
| kprobe_events   | 启用 kprobe 的配置        |
| uprobe_events	      | 启用 uprobe 的配置       |
| events ( 目录 )   | 事件（Event）跟踪器的控制文件： tracepoint、kprobe、uprobe        |
| trace      | 跟踪的输出 （Ring Buffer）       |
| trace_pipe   | 跟踪的输出；提供持续不断的数据流，适用于程序进行读取        |

2. 举例
- 内核函数调用跟踪
尝试对于内核中的系统调用函数`__x64_sys_openat`进行跟踪，可以跟踪的函数可以通过查看`/proc/kallsyms`文件确认。

如无特殊说明，都默认位于`/sys/kernel/debug/tracing`根目录。

```
# 使用 function 跟踪器，并将其设置到 current_tracer
$ echo function > current_tracer

# 将跟踪函数 __x64_sys_openat 设置到 set_ftrace_filter 文件中
$ echo __x64_sys_openat > set_ftrace_filter

# 开启全局的跟踪使能
$ echo 1 > tracing_on

# 运行 ls 命令触发 sys_openat 系统调用，新的内核版本中直接调用 sys_openat
$ ls -hl 

# 关闭
$ echo 0 > tracing_on

# 通过 cat trace 文件进行查看
$ cat trace
# tracer: function
#
# entries-in-buffer/entries-written: 176/176   #P:4
#
#                                _-----=> irqs-off
#                               / _----=> need-resched
#                              | / _---=> hardirq/softirq
#                              || / _--=> preempt-depth
#                              ||| /     delay
#           TASK-PID     CPU#  ||||   TIMESTAMP  FUNCTION
#              | |         |   ||||      |         |
              ls-2739    [002] ....  1822.837813: __x64_sys_openat <-do_syscall_64
              ls-2739    [002] ....  1822.837825: __x64_sys_openat <-do_syscall_64
              ls-2739    [002] ....  1822.837858: __x64_sys_openat <-do_syscall_64

$ echo nop > current_tracer

# 需要主要这里的 echo 后面有一个空格，即 “echo+ 空格>" 
$ echo  > set_ftrace_filter
```

- 函数被调用流程（栈）
有些场景我们更可能希望获取调用该内核函数的流程（即该函数是在何处被调用），这需要通过设置`options/func_stack_trace`选项实现。

```
$ echo function > current_tracer
$ echo __x64_sys_openat > set_ftrace_filter
$ echo 1 > options/func_stack_trace # 设置调用栈选项
$ echo 1 > tracing_on

$ ls -hl

$ echo 0 > tracing_on

$ cat trace
# tracer: function
#
# entries-in-buffer/entries-written: 152/152   #P:4
#
#                                _-----=> irqs-off
#                               / _----=> need-resched
#                              | / _---=> hardirq/softirq
#                              || / _--=> preempt-depth
#                              ||| /     delay
#           TASK-PID     CPU#  ||||   TIMESTAMP  FUNCTION
#              | |         |   ||||      |         |
              ls-5212    [001] ....  4988.551998: __x64_sys_openat <-do_syscall_64
              ls-5212    [001] ....  4988.552002: <stack trace>
 => __x64_sys_openat
 => do_syscall_64
 => entry_SYSCALL_64_after_hwframe

# 关闭
$ echo nop > current_tracer
$ echo  > set_ftrace_filter
$ echo 0 > options/func_stack_trace
```

- 函数调用子流程跟踪（栈）
如果想要分析内核函数调用的子流程（即本函数调用了哪些子函数，处理的流程如何），这时需要用到`function_graph`跟踪器，从字面意思就可看出这是函数调用关系跟踪。

```
# 将当前 current_tracer 设置为 function_graph
$ echo function_graph > current_tracer
$ echo __x64_sys_openat > set_graph_function

# 设置跟踪子函数的最大层级数
$ echo 3 > max_graph_depth  # 设置最大层级
$ echo 1 > tracing_on

$ ls -hl

$ echo 0 > tracing_on
#$ echo nop > set_graph_function
$ cat trace
# tracer: function_graph
#
# CPU  DURATION                  FUNCTION CALLS
# |     |   |                     |   |   |   |
   3)  systemd-526   =>    ls-5317
 ------------------------------------------

   3)               |  __x64_sys_openat() {
   3)               |    do_sys_open() {
   3)   0.750 us    |      getname();
   3)   0.422 us    |      get_unused_fd_flags();
   3)   4.470 us    |      do_filp_open();
   3)   0.229 us    |      __fsnotify_parent();
   3)   0.106 us    |      fsnotify();
   3)   0.128 us    |      fd_install();
   3)   0.170 us    |      putname();
   3)   7.375 us    |    }
   3)   8.766 us    |  }
```

- 内核跟踪点（tracepoint）跟踪

```
# available_events 文件中包括全部可用于跟踪的静态跟踪点
$ grep openat available_events
syscalls:sys_exit_openat
syscalls:sys_enter_openat

# 我们可以在 events/syscalls/sys_enter_openat 中查看该跟踪点相关的选项
$ ls -hl events/syscalls/sys_enter_openat
total 0
-rw-r--r-- 1 root root 0 Aug 14 06:12 enable # 是否启用跟踪 1 启用
-rw-r--r-- 1 root root 0 Aug 14 06:12 filter # 跟踪过滤
-r--r--r-- 1 root root 0 Aug 14 06:12 format # 跟踪点格式
-r--r--r-- 1 root root 0 Aug 14 06:12 hist
-r--r--r-- 1 root root 0 Aug 14 06:12 id
-rw-r--r-- 1 root root 0 Aug 14 06:12 trigger

$ cat events/syscalls/sys_enter_openat/format
name: sys_enter_openat
ID: 622
format:
        field:unsigned short common_type;       offset:0;       size:2; signed:0;
        field:unsigned char common_flags;       offset:2;       size:1; signed:0;
        field:unsigned char common_preempt_count;       offset:3;       size:1; signed:0;
        field:int common_pid;   offset:4;       size:4; signed:1;

        field:int __syscall_nr; offset:8;       size:4; signed:1;
        field:int dfd;  offset:16;      size:8; signed:0;
        field:const char * filename;    offset:24;      size:8; signed:0;
        field:int flags;        offset:32;      size:8; signed:0;
        field:umode_t mode;     offset:40;      size:8; signed:0;

print fmt: "dfd: 0x%08lx, filename: 0x%08lx, flags: 0x%08lx, mode: 0x%08lx", ((unsigned long)(REC->dfd)), ((unsigned long)(REC->filename)), ((unsigned long)(REC->flags)), ((unsigned long)(REC->mode))

# 使用 tracepoint 跟踪 sys_openat 系统调用
$ echo 1 > events/syscalls/sys_enter_openat/enable
$ echo 1 > tracing_on
$ cat trace
# tracer: nop
#
# entries-in-buffer/entries-written: 37/37   #P:4
#
#                                _-----=> irqs-off
#                               / _----=> need-resched
#                              | / _---=> hardirq/softirq
#                              || / _--=> preempt-depth
#                              ||| /     delay
#           TASK-PID     CPU#  ||||   TIMESTAMP  FUNCTION
#              | |         |   ||||      |         |
      irqbalance-940     [001] ....  5807.145124: sys_openat(dfd: ffffff9c, filename: 562bac4cf9f5, flags: 0, mode: 0)
      irqbalance-940     [001] ....  5807.145250: sys_openat(dfd: ffffff9c, filename: 562bac4cfa57, flags: 0, mode: 0)

# 关闭
$ echo 0 > tracing_on
$ echo 0 > events/syscalls/sys_enter_openat/enable
```

- kprobe 跟踪
kprobe 机制允许我们跟踪函数任意位置，还可用于获取函数参数与结果返回值。使用 kprobe 机制跟踪函数须是`available_filter_functions`列表中的子集。
kprobe 设置文件和相关文件如下所示，其中部分文件为设置 kprobe 跟踪函数后，Ftrace 自动创建：
+ `kprobe_events`：设置 kprobe 跟踪的事件属性
+ `kprobes/<GRP>/<EVENT>/enabled`：设置后动态生成，用于控制是否启用该内核函数的跟踪；
+ `kprobes/<GRP>/<EVENT>/filter`：设置后动态生成，kprobe 函数跟踪过滤器，与上述的跟踪点 fliter 类似；
+ `kprobes/<GRP>/<EVENT>/format`：设置后动态生成，kprobe 事件显示格式；
+ `kprobe_profile`：kprobe 事件统计性能数据
```
# p[:[GRP/]EVENT] [MOD:]SYM[+offs]|MEMADDR [FETCHARGS]
# GRP=my_grp EVENT=x64_sys_openat  
# SYM=__x64_sys_openat
# FETCHARGS = dfd=$arg1 flags=$arg3 mode=$arg4
$ echo 'p:my_grp/x64_sys_openat __x64_sys_openat dfd=$arg1 flags=$arg3 mode=$arg4' >> kprobe_events
$ cat events/my_grp/x64_sys_openat/format
name: x64_sys_openat
ID: 1698
format:
        field:unsigned short common_type;       offset:0;       size:2; signed:0;
        field:unsigned char common_flags;       offset:2;       size:1; signed:0;
        field:unsigned char common_preempt_count;       offset:3;       size:1; signed:0;
        field:int common_pid;   offset:4;       size:4; signed:1;

        field:unsigned long __probe_ip; offset:8;       size:8; signed:0;
        field:u64 dfd;  offset:16;      size:8; signed:0;
        field:u64 flags;        offset:24;      size:8; signed:0;
        field:u64 mode; offset:32;      size:8; signed:0;

print fmt: "(%lx) dfd=0x%Lx flags=0x%Lx mode=0x%Lx", REC->__probe_ip, REC->dfd, REC->flags, REC->mode

$ echo 1 > events/my_grp/x64_sys_openat/enable

$ cat trace
# tracer: nop
#
# entries-in-buffer/entries-written: 59/59   #P:4
#
#                                _-----=> irqs-off
#                               / _----=> need-resched
#                              | / _---=> hardirq/softirq
#                              || / _--=> preempt-depth
#                              ||| /     delay
#           TASK-PID     CPU#  ||||   TIMESTAMP  FUNCTION
#              | |         |   ||||      |         |
      irqbalance-940     [001] ....  5807.145124: sys_openat(dfd: ffffff9c, filename: 562bac4cf9f5, flags: 0, mode: 0)
      irqbalance-940     [001] ....  5807.145250: sys_openat(dfd: ffffff9c, filename: 562bac4cfa57, flags: 0, mode: 0)

# 关闭，注意需要先 echo 0 > enable 停止跟踪
# 然后再使用 "-:my_grp/x64_sys_openat" 停止，否则会正在使用或者忙的错误
$ echo 0 > events/my_grp/x64_sys_openat/enable
$ echo '-:my_grp/x64_sys_openat' >> kprobe_events
```
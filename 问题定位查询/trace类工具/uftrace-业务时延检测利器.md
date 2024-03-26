```table-of-contents
```
# 背景
# 应用
**偶发的超时突刺**出现在哪；**微突发**
# 使用
```bash
# /usr/local/uftrace/bin/uftrace -h
 uftrace -- function (graph) tracer for userspace

 usage: uftrace [COMMAND] [OPTION...] [<program>]

 COMMAND:
   record          Run a program and saves the trace data
   replay          Show program execution in the trace data
   report          Show performance statistics in the trace data
   live            Do record and replay in a row (default)
   info            Show system and program info in the trace data
   dump            Show low-level trace data
   recv            Save the trace data from network
   graph           Show function call graph in the trace data
   script          Run a script for recorded trace data
   tui             Show text user interface for graph and report

 OPTION:
      --avg-self             Show average/min/max of self function time
      --avg-total            Show average/min/max of total function time
  -a, --auto-args            Show arguments and return value of known functions
  -A, --argument=FUNC@arg[,arg,...]
                             Show function arguments
  -b, --buffer=SIZE          Size of tracing buffer (default: 128K)
      --chrome               Dump recorded data in chrome trace format
      --clock                Set clock source for timestamp (default: mono)
      --color=SET            Use color for output: yes, no, auto (default: auto)
      --column-offset=DEPTH  Offset of each column (default: 8)
      --column-view          Print tasks in separate columns
  -C, --caller-filter=FUNC   Only trace callers of those FUNCs
  -d, --data=DATA            Use this DATA instead of uftrace.data
      --debug-domain=DOMAIN  Filter debugging domain
      --demangle=TYPE        C++ symbol demangling: full, simple, no
                             (default: simple)
      --diff=DATA            Report differences
      --diff-policy=POLICY   Control diff report policy
                             (default: 'abs,compact,no-percent')
      --disable              Start with tracing disabled (deprecated)
  -D, --depth=DEPTH          Trace functions within DEPTH
  -e, --estimate-return      Use only entry record type for safety
      --event-full           Show all events outside of function
  -E, --Event=EVENT          Enable EVENT to save more information
      --flame-graph          Dump recorded data in FlameGraph format
      --flat                 Use flat output format
      --force                Trace even if executable is not instrumented
      --format=FORMAT        Use FORMAT for output: normal, html (default: normal)
  -f, --output-fields=FIELD  Show FIELDs in the replay or graph output
  -F, --filter=FUNC          Only trace those FUNCs
  -g  --agent                Start an agent in mcount to listen to commands
      --graphviz             Dump recorded data in DOT format
  -H, --hide=FUNC            Hide FUNCs from trace
      --host=HOST            Send trace data to HOST instead of write to file
  -k, --kernel               Trace kernel functions also (if supported)
      --keep-pid             Keep same pid during execution of traced program
      --kernel-buffer=SIZE   Size of kernel tracing buffer (default: 1408K)
      --kernel-full          Show kernel functions outside of user
      --kernel-only          Dump kernel data only
      --kernel-skip-out      Skip kernel functions outside of user (deprecated)
  -K, --kernel-depth=DEPTH   Trace kernel functions within DEPTH
      --libmcount-single     Use single thread version of libmcount
      --list-event           List available events
  -L, --loc-filter=LOCATION  Only trace functions in the source LOCATION
      --logfile=FILE         Save log messages to this file
  -l, --nest-libcall         Show nested library calls
      --libname              Show libname name with symbol name
      --libmcount-path=PATH  Load libmcount libraries from this PATH
      --match=TYPE           Support pattern match: regex, glob (default:
                             regex)
      --max-stack=DEPTH      Set max stack depth to DEPTH (default: 65535)
      --no-args              Do not show arguments and return value
      --no-comment           Don't show comments of returned functions
      --no-event             Disable (default) events
      --no-sched             Disable schedule events
      --no-sched-preempt     Hide pre-emptive schedule event
                             but show regular(sleeping) schedule event
      --no-libcall           Don't trace library function calls
      --no-merge             Don't merge leaf functions
      --no-pager             Do not use pager
      --no-pltbind           Do not bind dynamic symbols (LD_BIND_NOT)
      --no-randomize-addr    Disable ASLR (Address Space Layout Randomization)
      --nop                  No operation (for performance test)
      --num-thread=NUM       Create NUM recorder threads
  -N, --notrace=FUNC         Don't trace those FUNCs
      --opt-file=FILE        Read command-line options from FILE
  -p  --pid=PID              PID of an interactive mcount instance
      --port=PORT            Use PORT for network connection (default: 8090)
  -P, --patch=FUNC           Apply dynamic patching for FUNCs
      --record               Record a new trace data before running command
      --report               Show live report
      --rt-prio=PRIO         Record with real-time (FIFO) priority
  -r, --time-range=TIME~TIME Show output within the TIME(timestamp or elapsed time)
                             range only
      --run-cmd=CMDLINE      Command line that want to execute after tracing
                             data received
  -R, --retval=FUNC@retval   Show function return value
      --sample-time=TIME     Show flame graph with this sampling time
      --signal=SIG@act[,act,...]   Trigger action on those SIGnal
      --sort-column=INDEX    Sort diff report on column INDEX (default: 2)
      --srcline              Enable recording source line info
      --symbols              Print symbol tables
  -s, --sort=KEY[,KEY,...]   Sort reported functions by KEYs (default: 2)
  -S, --script=SCRIPT        Run a given SCRIPT in function entry and exit
  -t, --time-filter=TIME     Hide small functions run less than the TIME
      --task                 Show task info instead
      --task-newline         Interleave a newline when task is changed
      --tid=TID[,TID,...]    Only replay those tasks
      --time                 Print time information
      --trace=STATE          Set the recording state: on, off (default: on)
  -T, --trigger=FUNC@act[,act,...]
                             Trigger action on those FUNCs
  -U, --unpatch=FUNC         Don't apply dynamic patching for FUNCs
  -v, --debug                Print debug messages
      --verbose              Print verbose (debug) messages
      --with-syms=DIR        Use symbol files in the DIR
  -W, --watch=POINT          Watch and report POINT if it's changed
  -Z, --size-filter=SIZE     Apply dynamic patching for functions bigger than SIZE
  -h, --help                 Give this help list
      --usage                Give a short usage message
  -V, --version              Print program version

 Try `man uftrace [COMMAND]' for more information.
```
# 缺陷
![](attachments/Pasted%20image%2020240324220908.png)
目前还不支持对于一个已经运行的程序进行追踪。

# 其他

# 参考
```bash
# 业务时延检测利器-uftrace
https://www.cnblogs.com/t-bar/p/16898892.html
```
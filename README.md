 `plotman`: Chia自动P盘管理工具
这是用于操作Chia P图参数的管理工具,这个工具是在运行于P盘机上。具有以下功能：

This is a tool for managing [Chia](https://github.com/Chia-Network/chia-blockchain)
plotting operations.  The tool runs on the plotting machine and provides
the following functionality:

- 自动生成新的P图任务，可能会在多个临时目录里重叠，同时受全局速率和每个临时目录限制；
- Automatic spawning of new plotting jobs, possibly overlapping ("staggered")
  on multiple temp directories, rate-limited globally and by per-temp-dir
limits.
- 同步新生成的P图文件到远程主机上（农场/收割机），可以称之为“归档”;
- Rsync'ing of newly generated plots to a remote host (a farmer/harvester),
  called "archiving".
- 监视正在进行中的P图任务和归档任务、进度、使用资源状态、临时文件等；
- Monitoring of ongoing plotting and archiving jobs, progress, resources used,
  temp files, etc.
- 控制正在进行的绘图任务（挂起、恢复以及终止和清除临时文件）；
- Control of ongoing plotting jobs (suspend, resume, plus kill and clean up
  temp files).
- 该工具支持交互式实时仪表模式和命令行模式；
- Both an interactive live dashboard mode as well as command line mode tools.
- 分析完成任务的性能统计信息，以及汇总各种P图参数、临时目录类型；
- (very alpha) Analyzing performance statistics of past jobs, to aggregate on
  various plotting parameters or temp dir type.
  
Plotman设计使用以下配置：
Plotman is designed for the following configuration:

- 一台P盘设备应该有一组"tmp"目录、一个"tmp2"目录和一组"dst"目录去完成P图任务，其中：“dst"目录是用于P图时的临时缓存；
- A plotting machine with an array of `tmp` dirs, a single `tmp2` dir, and an
  array of `dst` dirs to which the plot jobs plot.  The `dst` dirs serve as a
temporary buffer space for generated plots.

- 一台拥有大量机械硬盘的充当农场的设备，可以通过rsyncd模块进行访问，并完成P图文件的数据填充，这些称之为“归档”目录；
- A farming machine with a large number of drives, made accessible via an
  `rsyncd` module, and to be entirely populated with plots.  These are known as
the `archive` directories.

在配置目录中P图任务使用STDOUT/STDERR重定向到日志文件，通过分析日志文件就可以对进程（P图阶段）、时序（性能分析）；
- Plot jobs are run with STDOUT/STDERR redirected to a log file in a configured
directory.  This allows analysis of progress (plot phase) as well as timing
(e.g. for analyzing performance).


## 功能性

Plotman是无状态工具，该工具不会保留P图任务已经启动的内部记录，而是依靠P图任务进程表，打开文件和日志文件来获取“正在发生的事情”，这意味着即使不同的登录会话中也可以停止和启动工具，而不会造成P图数据的信息丢失。

Plotman tools are stateless.  Rather than keep an internal record of what jobs
have been started, Plotman relies on the process tables, open files, and
logfiles of plot jobs to understand "what's going on".  This means the tools
can be stopped and started, even from a different login session, without loss
of information.  It also means Plotman can see and manage jobs started manually
or by other tools, as long as their STDOUT/STDERR redirected to a file in a
known logfile directory.  (Note: The tool relies on reading the chia plot
command line arguments and the format of the plot tool output.  Changes in
those may break this tool.)

Plot scheduling is done by waiting for a certain amount of wall time since the
last job was started, finding the best (e.g. least recently used) `tmp` dir for
plotting, and ensuring that job has progressed to at least a certain point
(e.g., phase 2, subphase 5).

Plots are output to the `dst` dirs, which serve as a temporary buffer until they
are rsync'd ("archived") to the farmer/harvester.  The archiver does several
things to attempt to avoid concurrent IO.  First, it only allows one rsync
process at a time (more sophisticated scheduling could remove this
restriction, but it's nontrivial).  Second, it inspects the pipeline of plot
jobs to see which `dst` dirs are about to have plots written to them.  This
is balanced against how full the `dst` drives are in a priority scheme.

It is, obviously, necessary that your rsync bandwidth exceeds your plotting
bandwidth.  Given this, in normal operation, the `dst` dirs remain empty until
a plot is finished, after which it is shortly thereafter picked up by the
archive job.  However, the decoupling provided by using `dst` drives as a
buffer means that should the farmer/harvester or the network become
unavailable, plotting continues uninterrupted.

## 屏幕截图概述


```
Plotman 19:01:06 (refresh 9s/20s)  |  Plotting: stagger (1623s/1800s) Archival: active pid 1599918
Prefixes:  tmp=/mnt/tmp  dst=/home/chia/chia/plots  archive=/plots (remote)

  #       plot id    k   tmp   dst    wall   phase    tmp       pid   stat      mem    user    sys     io               
  0   6b4e7375...   32    03   001    0:27     1:2    71G   1590196    SLP     5.5G    0:52   0:02     0s
  1   9ab50d0e...   32    02   005    1:00     1:4   199G   1539209    SLP     5.5G    3:50   0:09     0s
  2   018cf561...   32    01   000    1:32     1:5   224G   1530045    SLP     5.5G    4:46   0:11     2s
  3   f771de9c...   32    00   004    2:03     1:5   241G   1524772    SLP     5.5G    5:43   0:14     2s
...
 16   58045bef...   32    10   002   11:23     3:5   193G   1381622    RUN     5.4G   15:02   0:53   0:02
 17   8134a2dd...   32    11   003   11:55     3:6   148G   1372206    RUN     5.4G   15:27   0:57   0:03
 18   50165422...   32    08   001   12:43     3:6   102G   1357782    RUN     5.4G   16:14   1:00   0:03
 19   100df84f...   32    09   005   13:19     4:0      0   1347430    DSK   705.9M   16:44   1:04   0:06

tmp   ready    phases     tmp   ready    phases        dst   plots   GB free         phases         priority 
 00      --   1:5, 3:4     06      --   2:4            000   1       1890      1:5, 2:2, 3:4        47
 01      --   1:5, 3:4     07      --   2:2            001   0       1998      1:2, 1:7, 3:2, 3:6   34
 02      --   1:4, 3:3     08      --   1:7, 3:6       002   0       1967      1:6, 2:5, 3:5        42
 03      --   1:2, 3:2     09      --   2:1, 4:0       003   0       1998      1:6, 3:1, 3:6        34
 04      OK   3:1          10      --   1:6, 3:5       004   0       1998      1:5, 2:4, 3:4        46
 05      OK   2:5          11      --   1:6, 3:6       005   0       1955      1:4, 2:1, 3:3, 4:0   18

Archive dirs free space
000:   94GB | 005:   94GB | 012:   24GB | 017:   99GB | 022:   94GB | 027:   94GB | 032: 9998GB | 037: 9998GB
001:   94GB | 006:   93GB | 013:   25GB | 018:   94GB | 023:   94GB | 028:   94GB | 033: 9998GB |
002:   93GB | 009:   25GB | 014:   93GB | 019:   31GB | 024:   94GB | 029: 7777GB | 034: 9998GB |
003:   94GB | 010:   25GB | 015:   94GB | 020:   47GB | 025:   94GB | 030: 9998GB | 035: 9998GB |
004:   94GB | 011:   25GB | 016:   99GB | 021:   93GB | 026:   94GB | 031: 9998GB | 036: 9998GB |

Log:
01-02 18:33:53 Starting plot job: chia plots create -k 32 -r 8 -u 128 -b 4580 -t /mnt/tmp/03 -2 /mnt/tmp/a -d /home/chi
01-02 18:33:53 Starting archive: rsync --bwlimit=100000 --remove-source-files -P /home/chia/chia/plots/004/plot-k32-202
01-02 18:52:40 Starting archive: rsync --bwlimit=100000 --remove-source-files -P /home/chia/chia/plots/000/plot-k32-202
```
屏幕截图显示了Plotman的一些主要功能

The screenshot shows some of the main features of Plotman.

The first line shows the status.  The plotting status shows whether we just
started a plot, or, if not, why not (e.g., stagger time, tmp directories being
ready, etc.; in this case, the 1800s stagger between plots has not been reached
yet).  Archival status says whether we are currently archiving (and provides
the `rsync` pid) or whether there are no plots available in the `dst` drives to
archive.

The second line provides a key to some directory abbrevations used throughout.
For `tmp` and `dst` directories, we assume they have a common prefix, which is
computed and indicated here, after which they can be referred to (in context)
by their unique suffix.  For example, if we have `tmp` dirs `/mnt/tmp/00`,
`/mnt/tmp/01`, `/mnt/tmp/02`, etc., we show `/mnt/tmp` as the prefix here and
can then talk about `tmp` dirs `00` or `01` etc.  The `archive` directories are
the same except that these are paths on a remote host and accessed via an
`rsyncd` module (see `src/plotman/resources/plotman.yaml` for details).

The next table shows information about the active plotting jobs.  It is
abbreviated to show the most and least recently started jobs (the full list is
available via the command line mode).  It shows various information about the
plot jobs, including the plot ID (first 8 chars), the directories used,
walltime, the current plot phase and subphase, space used on the `tmp` drive,
pid, etc.

The next tables are a bit hard to read; there is actually a `tmp` table on the
left which is split into two tables for rendering purposes, and a `dst` table
on the right.  The `tmp` tables show the phases of the plotting jobs using
them, and whether or not they're ready to take a new plot job.  The `dst` table
shows how many plots have accumulated, how much free space is left, and the
phases of jobs that are destined to write to them, and finally, the priority
computed for the archive job to move the plots away.

The last table simply shows free space of drives on the remote
harverster/farmer.

Finally, the last section shows a log of actions performed -- namely, plot and
archive jobs initiated.  This is the one part of the interactive tool which is
stateful.  There is no permanent record of these executed command lines, so if
you start a new interactive plotman session, this log is empty.

## 局限性和问题

该工具仅在linux上进行了测试.
The system is tested on Linux only.  Plotman should be generalizable to other
platforms, but this is not done yet.  Some of the issues around making calls
out to command line programs (e.g., running `df` over `ssh` to obtain the free
space on the remote archive directories) are very linux-y.

交互模式使用了 "cures" 函数库，但是效果并不好。不能返回按键信息，屏幕无法调整大小，并且终端的最小尺寸也是非常的大。
The interactive mode uses the `curses` library ... poorly.  Keypresses are
not received, screen resizing does not work, and the minimum terminal size
is pretty big.

Plotman假设所有的P图文件都是K32；
Plotman assumes all plots are k32s.  Again, this is just an unimplemented
generalization.

Many features are inconsistently supported between either the "interactive"
mode or the command line mode.

还存在太多的bug和待处理事项；
There are many bugs and TODOs.

Plotman将会不断地在计算机的默认保存目录下检查"plotman.yaml"文件，需要生成默认的“plotman.yaml”，可以运行：
```shell
> plotman config generate
```

Plotman will always look for the `plotman.yaml` file within your computer at an OS-based
default location. To generate a default `plotman.yaml`, run:
```shell
> plotman config generate
```
显示当前默认位置下你的 `plotman.yaml`文件和检查 `plotman.yaml`是否存在，可以运行：
```shell
> plotman config path
```

To display the current location of your `plotman.yaml` file and check if it exists, run:
```shell
> plotman config path
```

([See also](https://github.com/ericaltendorf/plotman/pull/61#issuecomment-812967363)).

## 安装

Installation for Linux and macOS:

1. Plotman assumes that a functioning [Chia](https://github.com/Chia-Network/chia-blockchain)
   installation is present on the system.
      - virtual environment (Linux, macOS): Activate your `chia` environment by typing
        `source /path/to/your/chia/install/activate`.
      - dmg (macOS): Follow [these instructions](https://github.com/Chia-Network/chia-blockchain/wiki/CLI-Commands-Reference#mac)
        to add the `chia` binary to the `PATH`
2. Then, install Plotman using the following command:
   ```shell
    > pip install --force-reinstall git+https://github.com/ericaltendorf/plotman@main
    ```
3. Plotman will look for `plotman.yaml` within your computer at an OS-based
   default location. To create a default `plotman.yaml` and display its location,
   run the following command:
   ```shell
   > plotman config generate
   ```
   The default configuration file used as a starting point is located [here](./src/plotman/resources/plotman.yaml)
4. That's it! You can now run Plotman by typing `plotman version` to verify its version.
   Run `plotman --help` to learn about the available commands.

### Development note:

If you are forking Plotman, simply replace the installation step with `pip install --editable .[dev]` from the project root directory to install *your* version of plotman with test and development extras.

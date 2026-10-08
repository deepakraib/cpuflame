# cpuflame

`cpuflame` shows where a Linux program spends CPU time, and what to look at next.

It samples the process with `perf`, draws an interactive flame graph, and writes a short report. The width of each frame is on-CPU time in that function, including everything it called. Blue is the process name, orange is the kernel, and yellow is user code. Click a frame to zoom. Type to search.

The report names the frames that used the most CPU and gives a next step: missing symbols, a spinlock, the allocator, copies, kernel time, or no single hotspot. `cpuflame html` puts the graph and that conclusion on one page.

You do not need `flamegraph.pl`. `cpuflame` reads `perf.data` from the machine that recorded it, or a `perf.script` text file sent from another server. A `perf report` file is not enough.

Run the commands below from this repository. `tools/cpuflame` is a file in the repo, not a command installed on `PATH`.

```bash
git clone git@github.com:deepakraib/cpuflame.git
cd cpuflame
```

## Process

### 1) Install perf

On Ubuntu:

```bash
sudo apt-get install linux-tools-$(uname -r) linux-tools-generic -y
```

On RHEL and clones:

```bash
sudo yum install -y perf
```

`perf` must match the running kernel. Python 3 is also required.

`cpuflame check` says whether this user can record. If recording is blocked, it prints:

```bash
sudo sysctl kernel.perf_event_paranoid=1
```

`1` is enough to profile your own processes, including kernel stacks. Recording every process on the machine needs `0`, or run the capture under `sudo`.

### 2) Capture performance data with perf

In this example, perf captures data from the `mysqld` process for 60 seconds. Change `binary_to_monitor` to the program you want, such as `mongod`, `mysqld`, or `valkey-server`.

```bash
binary_to_monitor="mysqld"
sudo perf record -a -g -F99 -p $(pgrep -x ${binary_to_monitor}) -- sleep 60
```

`-a` collects samples from all CPU cores. `-g` records the call graph for the kernel and for user space. `-F99` collects 99 samples per second. `-p` limits the samples to that process.

The same capture, drawn as soon as it finishes:

```bash
binary_to_monitor="mysqld"
sudo python3 tools/cpuflame attach --binary "$binary_to_monitor" -d 60 -o flamegraph.svg
```

That runs `perf record -a -g -F 99 -p $(pgrep -x "$binary_to_monitor") -- sleep 60`, then writes `flamegraph.svg` and `flamegraph.txt`.

### 3) Convert the perf data to text

`perf record` writes a binary `perf.data` in the current directory. This turns it into text:

```bash
sudo perf script > perf.script
```

The text is readable, and it is the input for the flame graph. If you already have a `perf.script` file from another machine, start at the next step. You do not need `perf` for that.

### 4) Generate the flame graph

```bash
python3 tools/cpuflame html -i perf.script -o flamegraph.html
```

Open `flamegraph.html` in a browser. That is the report to read. The same command also writes `flamegraph.svg`.

`cpuflame html` works from `perf.script`, from `perf.data`, or from folded stacks:

```bash
python3 tools/cpuflame html -i perf.script -o flamegraph.html
python3 tools/cpuflame html -i perf.data -o flamegraph.html
```

The page has six parts:

1. **Sample count and the split.** How many on-CPU samples were captured, and what share is user code versus kernel.
2. **What is wrong.** One card per finding, largest first. Each card names the frame, the percent, the call path, and the next step. **Highlight in the graph** marks that frame in the picture.
3. **Conclusion.** Three columns: what is wrong, what to do, and anything unusual in this profile. If symbols are missing, the conclusion says the graph cannot be used to tune the program yet, and how to record again.
4. **Where the samples went.** Threads that hold at least 1% of the profile. Threads named `conn` plus a number are added together as connections.
5. **The flame graph.** The same interactive SVG. The bottom row is the root. Blue is the process, orange is the kernel, yellow is user code. Click a frame to zoom. Type to search. Esc clears.
6. **Widest frames and where the CPU actually was.** Inclusive time, then self time, each with the path from the root.

Nothing on the page is fetched from the network. Send `flamegraph.html` as a single file. The SVG beside it is the same picture without the written analysis.

Text-only outputs:

```bash
python3 tools/cpuflame graph -i perf.script -o flamegraph.svg
python3 tools/cpuflame report -i perf.script -o flamegraph.txt
```

From the binary `perf.data` file, skip step 3:

```bash
python3 tools/cpuflame graph -i perf.data -o flamegraph.svg
python3 tools/cpuflame report -i perf.data -o flamegraph.txt
```

Folded stacks also work. A line is a stack and a count:

```text
main;foo;bar 12
```

```bash
python3 tools/cpuflame graph -i stacks.txt -o flamegraph.svg
```

## How to read the graph

Width is on-CPU time in that function, including everything it called. The bottom row is the root. Stacks grow upward.

- Blue is the process name
- Orange is the kernel
- Yellow is user code

Click a frame to zoom. Type to search. Esc clears the search. Reset zoom is in the corner.

A wide parent is only a problem when its children do not already explain the time. Self time is the part of a frame not covered by its children. That is where the CPU was.

## The report

`app.txt` sits next to the SVG. It has four parts.

1. How to read the graph, in a few lines.
2. Where the samples went: user code, kernel, and the threads that hold at least 1% of the profile. Threads named `conn` plus a number are added together as connections. Idle (`swapper`, or `do_idle` when the process name is hidden) is listed and then left out of the rest. A frame is never shown above 100%.
3. The widest frames (inclusive time) and the frames with the most self time, each with the call path from the root. Frames that only pass the time to a single child are left off the widest list.
4. Findings, largest first. Each one names the frame, the percent, the path, and a next step.

A finding is raised only when the frame is a real share of the profile.

| What shows up | Next step |
| --- | --- |
| One function has at least 30% of self time | Optimize that function first |
| `malloc`, `free`, `operator new`, and the usual allocators | Cut the allocation rate. The caller is the first frame outside the allocator |
| `memcpy`, `memmove`, `memset` | Cut copies |
| Futexes, pthread locks, spinlocks, InnoDB `ut_delay` / `rw_lock_*_spin`, PostgreSQL `s_lock` / `LWLockAcquire` | The CPU is spinning. Threads blocked in a futex are off-CPU and are not in this profile. Shorten or split the critical section, and take an off-CPU profile to see the waiters |
| Page faults (`exc_page_fault` and the older `do_page_fault` entry) | The program is touching new memory. The hottest frame inside the fault is named |
| Kernel self time at least 20% | Named by the frame under the syscall (`vfs_read`, `tcp_sendmsg`), or as reclaim / transparent huge pages when the frames are `compact_zone`, `kswapd`, `shrink_*` |
| Compression, parsing, or regex | The cost is in that library. The hottest frame inside it and the caller above it are named |
| Python `_PyEval_EvalFrameDefault` | This is the eval loop, not your function. Re-record with `python -X perf` (3.12+) |
| A wide `[unknown]` | Symbols are missing, so the graph cannot be trusted yet. Java and Node builds point at a perf map |
| A stack 48 frames deep, or one name repeated 6 times | Recursion or a callback loop |
| Nothing above | No single CPU problem in this profile. If the time is spread out, the report names the deepest wide frame where the time actually splits |

## Folded stacks

Folded input still works. A line is a stack and a count:

```text
main;foo;bar 12
```

Files produced by `stackcollapse-perf.pl` mark kernel frames with `_[k]`. In those files the first frame is the process name. A plain file with no mark keeps every frame as user code, including `main`.

## Check the script

```bash
python3 tools/cpuflame self-test
```

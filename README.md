# cpuflame

Record a CPU profile with Linux `perf`, draw an interactive flame graph, and write a short report that names the hot frames and what to try next.

One command does all three steps. You do not need `flamegraph.pl`.

Run it from the repository root. `tools/cpuflame` is a file in this repo, not a command on `PATH`.

```bash
git clone git@github.com:deepakraib/cpuflame.git
cd cpuflame
python3 tools/cpuflame check
python3 tools/cpuflame run -o app.svg -- python3 app.py
```

That writes `app.svg` and `app.txt` in the current directory. Open the SVG in a browser.

## Requirements

- Linux, with a kernel that has performance events
- Python 3
- `perf` on `PATH`, built for the running kernel

`cpuflame check` says whether this user can record. If recording is blocked it prints the exact fix:

```bash
sudo sysctl kernel.perf_event_paranoid=1
```

`1` is enough to profile your own processes, including kernel stacks. `cpuflame system` needs `0`, or run `cpuflame` under `sudo`.

The package name depends on the distribution. On Debian and Ubuntu:

```bash
sudo apt install linux-tools-common linux-tools-$(uname -r)
```

On Fedora, RHEL, and SUSE, install the `perf` package that matches the running kernel. `perf` must match that kernel or recording fails.

macOS, Windows, and the BSDs cannot record. `example`, and `graph` / `report` on folded stacks or a `perf script` text file, need only Python 3.

## Commands

| Command | What it does |
| --- | --- |
| `cpuflame check` | Say whether this user can record |
| `cpuflame run -o app.svg -- cmd` | Record a command, write the SVG and the report |
| `cpuflame attach -p PID -d 20 -o proc.svg` | Record a running process (default 10 seconds) |
| `cpuflame system -d 10 -o system.svg` | Record every CPU |
| `cpuflame graph -i perf.data -o out.svg` | Draw the SVG only |
| `cpuflame report -i perf.data` | Write the report only |
| `cpuflame example -o example.svg` | Write a sample graph and report, no `perf` |
| `cpuflame self-test` | Check the parser, the SVG writer, and the report |

`run --duration` needs GNU `timeout` from coreutils. `attach` and `system` use `sleep`.

```bash
python3 tools/cpuflame run -d 15 -F 99 -o app.svg -- python3 app.py
python3 tools/cpuflame run --keep -o app.svg -- python3 app.py
python3 tools/cpuflame run --record-only -o app.svg -- python3 app.py
perf script -i perf.data | python3 tools/cpuflame graph -i - -o out.svg
```

`--keep` saves `perf.data` next to the SVG (`app.svg` becomes `app.perf.data`). `--record-only` saves `perf.data` and stops. `--no-report` writes the SVG only. `-o out` with no `.svg` writes the report to `out.txt`.

Useful flags:

- `-F`, `--freq` samples per second (default 99)
- `--call-graph fp|dwarf|lbr` stack unwinder (default `fp`). `lbr` is Intel-only
- `--comm NAME` keep one process (repeatable)
- `--no-comm` do not add the process name at the base of each stack
- `--top N` rows in the widest and self-time lists (default 8)
- `--width`, `--min-width`, `--icicle`, `--title`, `--open`

`--open` uses `xdg-open`. On a server the SVG is still written. Open it yourself.

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
2. Where the samples went: user code, kernel, and each process. Idle (`swapper`, or `do_idle` when the process name is hidden) is listed and then left out of the rest.
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

# gomon — a mini `htop` in Go

> **Level:** beginner · **Language:** Go · **Linux focus:** `/proc`, Docker, Compose, deploy
>
> Builds on [step 04 (the `/proc` virtual filesystem)](../../04-filesystem-hierarchy.md)
> and [step 11 (processes & `htop`)](../../11-processes.md). You'll write a small
> Go program that *is* a tiny `htop`, then package and deploy it.

This is a **build-along lesson**. Claude gives you the blocks of code and the
instructions; **you** create the files, run the commands, and read the output.
Build it one part at a time — don't paste everything at once. At the end there's
an appendix with the full files so you can diff against your own.

---

## What you'll build

A command called `gomon` that reads the Linux `/proc` filesystem and shows a live,
refreshing dashboard of the machine's health:

```
gomon — linux-lab   up 6h 06m   load 1.00 1.00 0.76

CPU  [|||||||||||         ] 53.8%
MEM  [|||                 ] 15.9%  (1244/7837 MB)

PID    RSS (MB)  COMMAND
1             5  gomon
42            4  bash
...

Ctrl+C to quit
```

The whole thing is **pure Go standard library** — no external packages. The trick
is that *all* the data already lives in files under `/proc`; we just read and parse
them. That's the "everything is a file" idea from step 01 made concrete.

It also has two non-interactive modes for scripting and servers:

```bash
gomon --json     # one JSON snapshot, then exit  (pipe to jq, scrape, alert)
gomon --once     # one frame, then exit
```

### Why this project

- **Cements `/proc`** — you'll read `/proc/uptime`, `/proc/loadavg`, `/proc/stat`,
  `/proc/meminfo`, and `/proc/<pid>/…` by hand and finally *see* what `htop` is doing.
- **Real Docker, not toy Docker** — a multi-stage build that produces a ~3 MB static
  binary on a minimal base image, run as a Compose service.
- **A deploy you can repeat** — two honest ways to put it on a server (Docker, and a
  systemd-managed binary), which previews steps 14 and 17.

---

## Before you start: the dev loop

`gomon` reads `/proc`, **which only exists on Linux.** Your Mac has no `/proc`, so
running it there just prints zeros. That's the lesson, not a bug — this is
Linux-specific software, so we develop the code on the Mac but **run it on Linux.**

You don't need Go installed on your Mac. The fastest loop is to run Go *inside* a
throwaway Linux container that bind-mounts your project folder:

```bash
# from inside this project folder (example-projects/gomon-beginner-go)
docker run --rm -it -v "$PWD":/app -w /app golang:1.26 go run .
```

- `-v "$PWD":/app` mounts your code into the container (the same bind-mount idea as
  the lab's `/work`, step 02).
- `-w /app` makes that the working directory.
- `golang:1.26` is the official Go image — it has the compiler so you don't have to.

Every time you want to run what you've built so far, re-run that command. (If you
*do* have Go on your Mac, `go build .` is a handy way to check it *compiles*, but it
won't show real data until it runs on Linux.)

---

## Part 1 — Scaffold the project

Create these files in this folder as we go. First, the module:

```bash
# module name + Go version — creates go.mod
#   (run on your Mac if you have Go, or inside the golang container)
go mod init gomon
```

If you don't have Go locally, just create **`go.mod`** by hand:

```
module gomon

go 1.26
```

We'll split the program into three files so each piece stays small:

- `collect.go` — read `/proc`, return a `Stats` struct (the data layer)
- `render.go` — turn a `Stats` into output (the view layer)
- `main.go` — flags + the refresh loop (the wiring)

---

## Part 2 — Read uptime and load average

Start `collect.go`. We'll define the data shape first, then fill it in piece by
piece. Begin with the two easiest files in `/proc`.

`/proc/uptime` is a single line — *seconds since boot*, then *idle seconds*:

```
$ cat /proc/uptime
21955.80 153402.11
```

`/proc/loadavg` is the numbers behind `uptime`'s "load average" — the 1, 5, and
15-minute run-queue averages (step 11):

```
$ cat /proc/loadavg
1.00 1.00 0.76 2/318 4567
```

Here's the start of `collect.go` — the struct plus those two readers:

```go
package main

import (
	"strconv"
	"strings"
	"os"
)

// Stats is one snapshot of the machine's health, built entirely from /proc.
type Stats struct {
	Hostname   string     `json:"hostname"`
	UptimeSec  float64    `json:"uptime_sec"`
	Load       [3]float64 `json:"load"`
	CPUPct     float64    `json:"cpu_pct"`
	MemUsedMB  int        `json:"mem_used_mb"`
	MemTotalMB int        `json:"mem_total_mb"`
	Procs      []Proc     `json:"top"`
}

// Proc is one process in the "top by memory" list.
type Proc struct {
	PID   int    `json:"pid"`
	Cmd   string `json:"cmd"`
	RSSMB int    `json:"rss_mb"`
}

func hostname() string {
	h, err := os.Hostname()
	if err != nil {
		return "unknown"
	}
	return h
}

// /proc/uptime → "12345.67 6789.01"; first field = seconds since boot.
func readUptime() float64 {
	b, err := os.ReadFile("/proc/uptime")
	if err != nil {
		return 0
	}
	fields := strings.Fields(string(b))
	if len(fields) == 0 {
		return 0
	}
	v, _ := strconv.ParseFloat(fields[0], 64)
	return v
}

// /proc/loadavg → "0.12 0.09 0.03 1/234 5678"; first three = 1/5/15-min load.
func readLoad() [3]float64 {
	var l [3]float64
	b, err := os.ReadFile("/proc/loadavg")
	if err != nil {
		return l
	}
	f := strings.Fields(string(b))
	for i := 0; i < 3 && i < len(f); i++ {
		l[i], _ = strconv.ParseFloat(f[i], 64)
	}
	return l
}
```

The pattern repeats for every metric: **`os.ReadFile` the `/proc` file →
`strings.Fields` to split on whitespace → `strconv` to parse the number.** That's
90% of this project. The `json:"..."` struct tags name the keys for `--json` mode later.

> **Note on the `_` in `v, _ := ...`** — these `/proc` files are kernel-formatted and
> effectively always parse, so we ignore the error and fall back to a zero value.
> In production code you'd handle it; here it keeps the example focused.

---

## Part 3 — Read memory

`/proc/meminfo` is a column of `Key: value kB` lines:

```
$ cat /proc/meminfo
MemTotal:        8024864 kB
MemFree:         1235440 kB
MemAvailable:    6580120 kB
Buffers:           82340 kB
Cached:          4920112 kB
...
```

The honest "memory in use" is **`MemTotal − MemAvailable`** — *not* `MemTotal −
MemFree`. `MemAvailable` is the kernel's own estimate of what programs can actually
get, counting reclaimable cache (this is exactly why `free -h` from step 11 shows a
separate "available" column). Add this to `collect.go`:

```go
import "bufio"  // add to the import block

// /proc/meminfo is "Key:   value kB" lines. Used = Total - Available.
func readMem() (usedMB, totalMB int) {
	f, err := os.Open("/proc/meminfo")
	if err != nil {
		return 0, 0
	}
	defer f.Close()

	var totalKB, availKB int
	sc := bufio.NewScanner(f)
	for sc.Scan() {
		line := sc.Text()
		switch {
		case strings.HasPrefix(line, "MemTotal:"):
			totalKB = kbValue(line)
		case strings.HasPrefix(line, "MemAvailable:"):
			availKB = kbValue(line)
		}
	}
	return (totalKB - availKB) / 1024, totalKB / 1024
}

// kbValue pulls the number out of a "MemTotal:   2031868 kB" line.
func kbValue(line string) int {
	f := strings.Fields(line)
	if len(f) < 2 {
		return 0
	}
	v, _ := strconv.Atoi(f[1])
	return v
}
```

Here we use `bufio.Scanner` instead of `ReadFile` because the file is many lines and
we only want two of them — scan line by line and pick the ones we need.

---

## Part 4 — CPU % (the interesting one)

CPU usage is **not** a number you can read in one shot. `/proc/stat`'s first line is
a set of *cumulative counters* — total time the CPU has spent in each state since
boot, measured in "jiffies":

```
$ cat /proc/stat
cpu  461420 12 119236 8930122 4821 0 9344 0 0 0
#    user   nice system idle    iowait irq softirq ...
```

To get "how busy *right now*", you read it **twice**, a moment apart, and look at how
much the counters moved:

```
busy% = (1 − idle_delta / total_delta) × 100
```

If between the two reads the idle counter grew by 80 and the total grew by 100, the
CPU was idle 80% of that window → **20% busy**. This is the same delta-sampling every
monitoring tool does (it's why `htop` needs a moment to show a number). Add to
`collect.go`:

```go
// cpuSample is a point-in-time reading of the aggregate CPU counters.
type cpuSample struct {
	idle, total uint64
}

// /proc/stat first line: "cpu user nice system idle iowait irq softirq ...".
// All counters are cumulative "jiffies" since boot.
func readCPUSample() cpuSample {
	b, err := os.ReadFile("/proc/stat")
	if err != nil {
		return cpuSample{}
	}
	line := strings.SplitN(string(b), "\n", 2)[0]
	f := strings.Fields(line)
	var s cpuSample
	for i := 1; i < len(f); i++ { // skip f[0] == "cpu"
		v, _ := strconv.ParseUint(f[i], 10, 64)
		s.total += v
		if i == 4 || i == 5 { // idle (f[4]) + iowait (f[5])
			s.idle += v
		}
	}
	return s
}

// cpuPct turns two cumulative samples into a busy-percentage for the interval
// between them: busy = 1 - idleDelta/totalDelta.
func cpuPct(prev, cur cpuSample) float64 {
	dTotal := float64(cur.total - prev.total)
	dIdle := float64(cur.idle - prev.idle)
	if dTotal <= 0 {
		return 0
	}
	return (1 - dIdle/dTotal) * 100
}
```

This is the one place where *time* matters, and it shapes the whole program: the main
loop will keep the previous sample around and feed both into `cpuPct`.

---

## Part 5 — Top processes

Every running process has a directory `/proc/<pid>/`. Listing `/proc` and keeping the
**numeric** entries gives you the process table — that's literally what `ps` and
`pgrep` (step 11) do under the hood:

```
$ ls /proc
1  42  88  131  cpuinfo  meminfo  stat  uptime  loadavg  self  ...
#  ^pids^                ^^ non-numeric: kernel files, skip these
```

For each PID we read two tiny files:
- `/proc/<pid>/comm` — the command name (e.g. `bash`). *Capped at 15 chars by the
  kernel* — don't be surprised when long names get clipped.
- `/proc/<pid>/statm` — memory in **pages**; field 2 is the *resident set size* (RAM
  actually in use). Multiply by the page size to get bytes.

Add the scanner to `collect.go`:

```go
import (
	"path/filepath"
	"sort"
)

// readProcs scans /proc/<pid>/ for every process and returns the top n by RSS.
func readProcs(n int) []Proc {
	entries, err := os.ReadDir("/proc")
	if err != nil {
		return nil
	}
	pageBytes := int64(os.Getpagesize())

	var procs []Proc
	for _, e := range entries {
		pid, err := strconv.Atoi(e.Name())
		if err != nil {
			continue // not a numeric PID dir (e.g. "self", "meminfo")
		}
		// /proc/<pid>/statm field 2 = resident pages.
		b, err := os.ReadFile(filepath.Join("/proc", e.Name(), "statm"))
		if err != nil {
			continue // the process may have exited mid-scan — skip it
		}
		f := strings.Fields(string(b))
		if len(f) < 2 {
			continue
		}
		rssPages, _ := strconv.ParseInt(f[1], 10, 64)

		cb, _ := os.ReadFile(filepath.Join("/proc", e.Name(), "comm"))

		procs = append(procs, Proc{
			PID:   pid,
			Cmd:   strings.TrimSpace(string(cb)),
			RSSMB: int(rssPages * pageBytes / (1024 * 1024)),
		})
	}

	sort.Slice(procs, func(i, j int) bool { return procs[i].RSSMB > procs[j].RSSMB })
	if len(procs) > n {
		procs = procs[:n]
	}
	return procs
}
```

Two real-world details baked in: processes can **exit mid-scan**, so a failed read is
*skipped*, not fatal; and we **sort by RSS descending** then keep the top `n`. (Sorting
by memory is the simplest start — per-process *CPU%* needs the same two-sample delta
trick as Part 4, and it's the first stretch goal below.)

Finally, tie the layer together with one `collect` function at the top of `collect.go`
(just under the `Proc` type):

```go
func collect(prev, cur cpuSample) Stats {
	used, total := readMem()
	return Stats{
		Hostname:   hostname(),
		UptimeSec:  readUptime(),
		Load:       readLoad(),
		CPUPct:     cpuPct(prev, cur),
		MemUsedMB:  used,
		MemTotalMB: total,
		Procs:      readProcs(8),
	}
}
```

---

## Part 6 — Render and the main loop

Now `render.go` (the view) and `main.go` (the wiring). The neat trick here: the same
binary should be a **live full-screen monitor** when you run it in a terminal, but emit
**one log line per reading** when its output is piped (like in `docker compose logs`).
We detect that by asking whether stdout is a TTY — exactly how `ls` only adds colour
when talking to a terminal.

`render.go`:

```go
package main

import (
	"fmt"
	"os"
	"strings"
	"time"
)

// render picks output for where it runs: a live full-screen view on a terminal,
// or one log line per frame when output is a pipe/file (e.g. docker compose logs).
func render(s Stats) {
	if isTTY() {
		renderTUI(s)
	} else {
		renderLine(s)
	}
}

func isTTY() bool {
	fi, err := os.Stdout.Stat()
	if err != nil {
		return false
	}
	return fi.Mode()&os.ModeCharDevice != 0
}

func renderTUI(s Stats) {
	fmt.Print("\033[H\033[2J") // ANSI: cursor home + clear screen
	fmt.Printf("gomon — %s   up %s   load %.2f %.2f %.2f\n\n",
		s.Hostname, humanUptime(s.UptimeSec), s.Load[0], s.Load[1], s.Load[2])

	fmt.Printf("CPU  %s %5.1f%%\n", bar(s.CPUPct), s.CPUPct)
	fmt.Printf("MEM  %s %5.1f%%  (%d/%d MB)\n\n", bar(memPct(s)), memPct(s), s.MemUsedMB, s.MemTotalMB)

	fmt.Printf("%-6s %9s  %s\n", "PID", "RSS (MB)", "COMMAND")
	for _, p := range s.Procs {
		fmt.Printf("%-6d %9d  %s\n", p.PID, p.RSSMB, p.Cmd)
	}
	fmt.Print("\nCtrl+C to quit\n")
}

func renderLine(s Stats) {
	fmt.Printf("%s host=%s cpu=%.1f%% mem=%.1f%% (%d/%d MB) load=%.2f\n",
		time.Now().Format(time.RFC3339), s.Hostname,
		s.CPUPct, memPct(s), s.MemUsedMB, s.MemTotalMB, s.Load[0])
}

func memPct(s Stats) float64 {
	if s.MemTotalMB <= 0 {
		return 0
	}
	return float64(s.MemUsedMB) / float64(s.MemTotalMB) * 100
}

// bar draws a 20-wide [|||   ] meter for a 0..100 percentage.
func bar(pct float64) string {
	const width = 20
	filled := int(pct / 100 * width)
	if filled < 0 {
		filled = 0
	}
	if filled > width {
		filled = width
	}
	return "[" + strings.Repeat("|", filled) + strings.Repeat(" ", width-filled) + "]"
}

func humanUptime(sec float64) string {
	d := time.Duration(sec) * time.Second
	return fmt.Sprintf("%dh %02dm", int(d.Hours()), int(d.Minutes())%60)
}
```

The `\033[H\033[2J` is an **ANSI escape**: `\033` is the Escape character, `[H` moves
the cursor to the top-left, `[2J` clears the screen. Printing it each frame is what
makes the display "refresh in place."

`main.go`:

```go
package main

import (
	"encoding/json"
	"flag"
	"os"
	"time"
)

func main() {
	interval := flag.Duration("interval", 2*time.Second, "refresh interval between frames")
	jsonOut := flag.Bool("json", false, "print one JSON snapshot and exit")
	once := flag.Bool("once", false, "print one text frame and exit")
	flag.Parse()

	// CPU% is a rate: it needs two readings of /proc/stat separated by time.
	// Grab the first reading now; every frame compares against the previous one.
	prev := readCPUSample()

	if *jsonOut {
		s, _ := snapshot(prev, 300*time.Millisecond)
		enc := json.NewEncoder(os.Stdout)
		enc.SetIndent("", "  ")
		_ = enc.Encode(s)
		return
	}
	if *once {
		s, _ := snapshot(prev, 300*time.Millisecond)
		render(s)
		return
	}

	for {
		var s Stats
		s, prev = snapshot(prev, *interval)
		render(s)
	}
}

// snapshot waits, takes a fresh CPU sample, and collects a full Stats frame.
// It returns the new CPU sample so the caller can reuse it as the next "prev".
func snapshot(prev cpuSample, wait time.Duration) (Stats, cpuSample) {
	time.Sleep(wait)
	cur := readCPUSample()
	return collect(prev, cur), cur
}
```

The loop is deliberately simple: wait an interval, sample, render, repeat — and carry
`prev` forward so each CPU% covers the gap since the last frame. `flag` (Go's stdlib
arg parser) gives us `--interval`, `--json`, and `--once` for free.

---

## Part 7 — Run it

From this folder, in the Linux Go container:

```bash
docker run --rm -it -v "$PWD":/app -w /app golang:1.26 go run .
```

You should get the live dashboard, refreshing every 2s. `Ctrl+C` to quit. Try the
other modes (note `--json` is piped, so the TTY check sends it to JSON correctly):

```bash
docker run --rm -v "$PWD":/app -w /app golang:1.26 go run . --json
docker run --rm -v "$PWD":/app -w /app golang:1.26 go run . --once --interval=500ms
```

Expected `--json` (your numbers will differ):

```json
{
  "hostname": "…",
  "uptime_sec": 21955.8,
  "load": [1, 1, 0.76],
  "cpu_pct": 53.8,
  "mem_used_mb": 1244,
  "mem_total_mb": 7837,
  "top": [{ "pid": 1, "cmd": "gomon", "rss_mb": 5 }]
}
```

**You'll only see one or two processes.** That's expected and important: a container
has its own **PID namespace**, so it only sees *its own* processes (step 21). We fix
that with `pid: host` in Compose (Part 9). Pipe JSON through `jq` to prove it's real
structured data:

```bash
docker run --rm -v "$PWD":/app -w /app golang:1.26 go run . --json | docker run --rm -i ghcr.io/jqlang/jq '.mem_used_mb'
```

---

## Part 8 — Containerize (multi-stage, static, tiny)

Now package `gomon` into its own image. Create **`Dockerfile`**:

```dockerfile
# ---- build stage: compile inside the full Go toolchain image ----
FROM golang:1.26 AS build
WORKDIR /src
COPY go.mod ./
COPY *.go ./
# CGO_ENABLED=0 → a fully static binary with no libc dependency,
# so it can run on a near-empty base image.
RUN CGO_ENABLED=0 GOOS=linux go build -o /gomon .

# ---- run stage: copy just the binary onto a minimal base ----
FROM gcr.io/distroless/static-debian12
COPY --from=build /gomon /gomon
ENTRYPOINT ["/gomon"]
```

This is the **multi-stage build** pattern (step 21): the heavy `golang` image (~800 MB)
is used only to *compile*; the final image is the tiny `distroless/static` base plus
your ~3 MB binary. Distroless has **no shell and no package manager** — a smaller attack
surface, and nothing to exploit if someone breaks in. Build and run it:

```bash
docker build -t gomon .
docker run --rm gomon --once          # one log line (no TTY)
docker run --rm -it gomon             # live view (TTY allocated by -it)
```

> Reads `/proc` only — it never writes anything and needs no privileges, so this is
> a genuinely safe little container to run anywhere.

---

## Part 9 — Run it with Docker Compose

Create **`docker-compose.yml`** so `gomon` runs as a managed service:

```yaml
# gomon as a tiny monitoring service.
#   docker compose up -d --build     # build + start; emits one line per interval
#   docker compose logs -f gomon     # watch the readings (the structured log lines)
#   docker compose run --rm gomon    # interactive full-screen view (TTY)
#   docker compose down              # stop & remove
services:
  gomon:
    build: .
    image: gomon
    container_name: gomon
    restart: unless-stopped
    # Show the HOST's processes, not just this container's one PID.
    # (Removes the PID-namespace isolation — see the note below.)
    pid: host
    command: ["--interval=10s"]
```

```bash
docker compose up -d --build
docker compose logs -f gomon
```

Because Compose captures stdout through a pipe (not a TTY), `gomon` automatically uses
its **log-line** mode — one structured line every 10s, perfect for `docker compose logs`:

```
gomon | 2026-06-09T12:45:46Z host=linux-lab cpu=4.7% mem=16.1% (1258/7837 MB) load=0.08
gomon | 2026-06-09T12:45:56Z host=linux-lab cpu=5.1% mem=16.2% (1262/7837 MB) load=0.11
```

For the **interactive** dashboard, `docker compose run` allocates a TTY, so the same
binary draws the full-screen view:

```bash
docker compose run --rm gomon --interval=1s
```

### The `pid: host` decision

- **Without `pid: host`** the container sees only its own process (the PID 1 you saw in
  Part 7) — useless for monitoring the machine.
- **With `pid: host`** the container shares the host's PID namespace, so the top-processes
  list shows everything on the box — what you actually want from a monitor.
- Trade-off: that's a real reduction in isolation, so you grant it *only* to a tool that
  genuinely needs to see host processes. (CPU/memory/load already reflect the host even
  without it — only the *process list* is namespaced.)

---

## Part 10 — Install it on a server

Two honest paths. Path A matches the Docker-centric arc of this course; Path B is the
classic "ship a binary" approach and previews steps 14 (systemd) and 17 (SSH).

### Path A — Docker (recommended)

Copy the project to the server and bring it up there. (`scp`/`ssh` are step 17 — for
now, the shapes:)

```bash
# on your Mac — copy the project folder to the server
scp -r . youruser@your-server:/opt/gomon
#   …or on the server:  git clone <your-repo> /opt/gomon

# on the server
ssh youruser@your-server
cd /opt/gomon
docker compose up -d --build      # Docker builds for the server's own CPU arch
docker compose logs -f gomon      # confirm readings
```

Because the build happens *on the server*, you never worry about CPU architecture —
Docker compiles for whatever the server is. `restart: unless-stopped` means it comes
back after a reboot. To update later: `git pull` (or re-`scp`), then
`docker compose up -d --build`.

### Path B — a static binary + systemd

No Docker on the server? Ship the single static binary and let **systemd** keep it
running (this is exactly step 14, applied):

```bash
# on your Mac / CI — cross-compile for the server's architecture.
#   Most cloud VMs are amd64; ARM VMs (Graviton, Ampere) use arm64.
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o gomon .

# ship the binary
scp gomon youruser@your-server:/tmp/gomon
ssh youruser@your-server 'sudo install -m 0755 /tmp/gomon /usr/local/bin/gomon'
```

On the server, create the unit at `/etc/systemd/system/gomon.service`:

```ini
[Unit]
Description=gomon system monitor
After=network.target

[Service]
ExecStart=/usr/local/bin/gomon --interval=30s
Restart=always
# It only reads /proc, so run it as an unprivileged throwaway user (least privilege).
DynamicUser=yes

[Install]
WantedBy=multi-user.target
```

Then enable and watch it — its log lines go straight to the journal:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now gomon     # start now + on every boot
journalctl -u gomon -f                # follow its output (the container-free tail -f)
```

> Heads-up: the lab container in `practice/` doesn't run systemd as PID 1, so Path B
> is for a real VM or a systemd-enabled box (see [step 14](../../14-services-systemd.md)).
> Path A works in the lab today.

---

## Exercises

1. **Build it part by part** as above. After each part, re-run in the Go container and
   check the new field shows up. Don't move on until a part runs.
2. **Make it sweat.** In the live view, open a second shell into the same container and
   run `yes > /dev/null &` (the step 11 runaway-CPU drill). Watch `gomon`'s CPU bar climb,
   then `kill %1` and watch it fall. You just used your own tool to do step 11's exercise.
3. **Prove the PID namespace.** Run the container *without* `pid: host` and count the
   processes; add `pid: host`, restart, and count again. Explain the difference to Claude.
4. **Verify "static."** `docker run --rm gomon --once` works on `distroless/static`, which
   has no libc. Rebuild the Dockerfile's run stage `FROM scratch` (truly empty) and confirm
   it *still* runs — that's what `CGO_ENABLED=0` bought you.
5. **Scrape it.** Pipe `--json` through `jq` to print just `.cpu_pct`, then write a
   one-line shell loop that samples it every 5s into a file (reuse `>>` from step 08).

## Stretch goals / ideas to extend

- **Per-process CPU%** — apply Part 4's two-sample delta to each PID (`utime`+`stime`,
  fields 14–15 of `/proc/<pid>/stat`), and add a flag to sort by CPU vs memory.
- **Disk usage** — add a `df`-style reading (`syscall.Statfs` on `/`) to the dashboard.
- **Colour thresholds** — turn the CPU/MEM bar red past 80% with ANSI colour codes.
- **Network** — parse `/proc/net/dev` for per-interface RX/TX byte counters (another delta).
- **A `/metrics` endpoint** — add a tiny `net/http` server exposing the same `Stats` in
  Prometheus format. (That's the "exporter" project — a natural sequel once you've done
  the web-server lessons 22–24.)
- **Full process name** — read `/proc/<pid>/cmdline` (NUL-separated, full path) instead of
  the 15-char `comm`.

When you finish, ask Claude to review your code, and log it in
[PROGRESS.md](../../PROGRESS.md).

---

## Best practices & pitfalls

- **CPU% needs two samples.** A single read of `/proc/stat` can't give you a percentage —
  it's a *rate*. If your CPU shows 0% or 100% constantly, you're not keeping `prev` between
  frames, or your interval is ~0.
- **Used memory = Total − Available**, not Total − Free. Using `MemFree` makes a healthy
  machine look alarmingly full, because Linux deliberately uses spare RAM for cache.
- **Processes vanish mid-scan.** Between `ls /proc` and reading `/proc/<pid>/statm`, the
  process can exit. *Skip* read errors per-PID; don't abort the whole scan.
- **`pid: host` is a privilege.** Grant it only because a monitor genuinely needs host
  visibility, and understand it drops PID isolation. Don't sprinkle it on other containers.
- **`/proc` is Linux-only.** Develop on the Mac, but always *run* on Linux. Empty output on
  macOS isn't a bug in your code — there's no `/proc` to read.
- **Static binary = small, portable image.** `CGO_ENABLED=0` + multi-stage + a `static`/
  `scratch` base is the idiomatic way to ship Go. Don't deploy the 800 MB `golang` image.

## Checklist

- [ ] I can explain what `/proc/uptime`, `/proc/loadavg`, `/proc/stat`, `/proc/meminfo`,
      and `/proc/<pid>/{comm,statm}` each contain.
- [ ] I understand why CPU% requires two samples over time (delta of cumulative counters).
- [ ] I built and ran `gomon` on Linux, in all three modes (live / `--json` / `--once`).
- [ ] I wrote a multi-stage Dockerfile that produces a static binary on a minimal base.
- [ ] I ran it as a Compose service, watched the log lines, and understand `pid: host`.
- [ ] I can describe two ways to deploy it to a server (Docker, and binary + systemd).

## Resources

- The `proc` filesystem (every file we read): https://man7.org/linux/man-pages/man5/proc.5.html
- Go `os`/`bufio`/`strconv` stdlib: https://pkg.go.dev/os · https://pkg.go.dev/bufio · https://pkg.go.dev/strconv
- Go `flag` package: https://pkg.go.dev/flag
- Distroless base images: https://github.com/GoogleContainerTools/distroless
- Multi-stage Docker builds: https://docs.docker.com/build/building/multi-stage/

---

## Appendix — full files (for checking your work)

Build it yourself first; use these only to diff. Four files in this folder:
`go.mod`, `collect.go`, `render.go`, `main.go`.

<details>
<summary><code>go.mod</code></summary>

```
module gomon

go 1.26
```
</details>

<details>
<summary><code>collect.go</code></summary>

```go
package main

import (
	"bufio"
	"os"
	"path/filepath"
	"sort"
	"strconv"
	"strings"
)

// Stats is one snapshot of the machine's health, built entirely from /proc.
type Stats struct {
	Hostname   string     `json:"hostname"`
	UptimeSec  float64    `json:"uptime_sec"`
	Load       [3]float64 `json:"load"`
	CPUPct     float64    `json:"cpu_pct"`
	MemUsedMB  int        `json:"mem_used_mb"`
	MemTotalMB int        `json:"mem_total_mb"`
	Procs      []Proc     `json:"top"`
}

// Proc is one process in the "top by memory" list.
type Proc struct {
	PID   int    `json:"pid"`
	Cmd   string `json:"cmd"`
	RSSMB int    `json:"rss_mb"`
}

func collect(prev, cur cpuSample) Stats {
	used, total := readMem()
	return Stats{
		Hostname:   hostname(),
		UptimeSec:  readUptime(),
		Load:       readLoad(),
		CPUPct:     cpuPct(prev, cur),
		MemUsedMB:  used,
		MemTotalMB: total,
		Procs:      readProcs(8),
	}
}

func hostname() string {
	h, err := os.Hostname()
	if err != nil {
		return "unknown"
	}
	return h
}

// /proc/uptime → "12345.67 6789.01"; first field = seconds since boot.
func readUptime() float64 {
	b, err := os.ReadFile("/proc/uptime")
	if err != nil {
		return 0
	}
	fields := strings.Fields(string(b))
	if len(fields) == 0 {
		return 0
	}
	v, _ := strconv.ParseFloat(fields[0], 64)
	return v
}

// /proc/loadavg → "0.12 0.09 0.03 1/234 5678"; first three = 1/5/15-min load.
func readLoad() [3]float64 {
	var l [3]float64
	b, err := os.ReadFile("/proc/loadavg")
	if err != nil {
		return l
	}
	f := strings.Fields(string(b))
	for i := 0; i < 3 && i < len(f); i++ {
		l[i], _ = strconv.ParseFloat(f[i], 64)
	}
	return l
}

// /proc/meminfo is "Key:   value kB" lines. Used = Total - Available.
func readMem() (usedMB, totalMB int) {
	f, err := os.Open("/proc/meminfo")
	if err != nil {
		return 0, 0
	}
	defer f.Close()

	var totalKB, availKB int
	sc := bufio.NewScanner(f)
	for sc.Scan() {
		line := sc.Text()
		switch {
		case strings.HasPrefix(line, "MemTotal:"):
			totalKB = kbValue(line)
		case strings.HasPrefix(line, "MemAvailable:"):
			availKB = kbValue(line)
		}
	}
	return (totalKB - availKB) / 1024, totalKB / 1024
}

// kbValue pulls the number out of a "MemTotal:   2031868 kB" line.
func kbValue(line string) int {
	f := strings.Fields(line)
	if len(f) < 2 {
		return 0
	}
	v, _ := strconv.Atoi(f[1])
	return v
}

// cpuSample is a point-in-time reading of the aggregate CPU counters.
type cpuSample struct {
	idle, total uint64
}

// /proc/stat first line: "cpu user nice system idle iowait irq softirq ...".
// All counters are cumulative "jiffies" since boot.
func readCPUSample() cpuSample {
	b, err := os.ReadFile("/proc/stat")
	if err != nil {
		return cpuSample{}
	}
	line := strings.SplitN(string(b), "\n", 2)[0]
	f := strings.Fields(line)
	var s cpuSample
	for i := 1; i < len(f); i++ { // skip f[0] == "cpu"
		v, _ := strconv.ParseUint(f[i], 10, 64)
		s.total += v
		if i == 4 || i == 5 { // idle (f[4]) + iowait (f[5])
			s.idle += v
		}
	}
	return s
}

// cpuPct turns two cumulative samples into a busy-percentage for the interval
// between them: busy = 1 - idleDelta/totalDelta.
func cpuPct(prev, cur cpuSample) float64 {
	dTotal := float64(cur.total - prev.total)
	dIdle := float64(cur.idle - prev.idle)
	if dTotal <= 0 {
		return 0
	}
	return (1 - dIdle/dTotal) * 100
}

// readProcs scans /proc/<pid>/ for every process and returns the top n by RSS.
func readProcs(n int) []Proc {
	entries, err := os.ReadDir("/proc")
	if err != nil {
		return nil
	}
	pageBytes := int64(os.Getpagesize())

	var procs []Proc
	for _, e := range entries {
		pid, err := strconv.Atoi(e.Name())
		if err != nil {
			continue // not a numeric PID dir (e.g. "self", "meminfo")
		}
		// /proc/<pid>/statm field 2 = resident pages.
		b, err := os.ReadFile(filepath.Join("/proc", e.Name(), "statm"))
		if err != nil {
			continue // the process may have exited mid-scan — skip it
		}
		f := strings.Fields(string(b))
		if len(f) < 2 {
			continue
		}
		rssPages, _ := strconv.ParseInt(f[1], 10, 64)

		cb, _ := os.ReadFile(filepath.Join("/proc", e.Name(), "comm"))

		procs = append(procs, Proc{
			PID:   pid,
			Cmd:   strings.TrimSpace(string(cb)),
			RSSMB: int(rssPages * pageBytes / (1024 * 1024)),
		})
	}

	sort.Slice(procs, func(i, j int) bool { return procs[i].RSSMB > procs[j].RSSMB })
	if len(procs) > n {
		procs = procs[:n]
	}
	return procs
}
```
</details>

<details>
<summary><code>render.go</code></summary>

```go
package main

import (
	"fmt"
	"os"
	"strings"
	"time"
)

// render picks output for where it runs: a live full-screen view on a terminal,
// or one log line per frame when output is a pipe/file (e.g. docker compose logs).
func render(s Stats) {
	if isTTY() {
		renderTUI(s)
	} else {
		renderLine(s)
	}
}

func isTTY() bool {
	fi, err := os.Stdout.Stat()
	if err != nil {
		return false
	}
	return fi.Mode()&os.ModeCharDevice != 0
}

func renderTUI(s Stats) {
	fmt.Print("\033[H\033[2J") // ANSI: cursor home + clear screen
	fmt.Printf("gomon — %s   up %s   load %.2f %.2f %.2f\n\n",
		s.Hostname, humanUptime(s.UptimeSec), s.Load[0], s.Load[1], s.Load[2])

	fmt.Printf("CPU  %s %5.1f%%\n", bar(s.CPUPct), s.CPUPct)
	fmt.Printf("MEM  %s %5.1f%%  (%d/%d MB)\n\n", bar(memPct(s)), memPct(s), s.MemUsedMB, s.MemTotalMB)

	fmt.Printf("%-6s %9s  %s\n", "PID", "RSS (MB)", "COMMAND")
	for _, p := range s.Procs {
		fmt.Printf("%-6d %9d  %s\n", p.PID, p.RSSMB, p.Cmd)
	}
	fmt.Print("\nCtrl+C to quit\n")
}

func renderLine(s Stats) {
	fmt.Printf("%s host=%s cpu=%.1f%% mem=%.1f%% (%d/%d MB) load=%.2f\n",
		time.Now().Format(time.RFC3339), s.Hostname,
		s.CPUPct, memPct(s), s.MemUsedMB, s.MemTotalMB, s.Load[0])
}

func memPct(s Stats) float64 {
	if s.MemTotalMB <= 0 {
		return 0
	}
	return float64(s.MemUsedMB) / float64(s.MemTotalMB) * 100
}

// bar draws a 20-wide [|||   ] meter for a 0..100 percentage.
func bar(pct float64) string {
	const width = 20
	filled := int(pct / 100 * width)
	if filled < 0 {
		filled = 0
	}
	if filled > width {
		filled = width
	}
	return "[" + strings.Repeat("|", filled) + strings.Repeat(" ", width-filled) + "]"
}

func humanUptime(sec float64) string {
	d := time.Duration(sec) * time.Second
	return fmt.Sprintf("%dh %02dm", int(d.Hours()), int(d.Minutes())%60)
}
```
</details>

<details>
<summary><code>main.go</code></summary>

```go
package main

import (
	"encoding/json"
	"flag"
	"os"
	"time"
)

func main() {
	interval := flag.Duration("interval", 2*time.Second, "refresh interval between frames")
	jsonOut := flag.Bool("json", false, "print one JSON snapshot and exit")
	once := flag.Bool("once", false, "print one text frame and exit")
	flag.Parse()

	// CPU% is a rate: it needs two readings of /proc/stat separated by time.
	// Grab the first reading now; every frame compares against the previous one.
	prev := readCPUSample()

	if *jsonOut {
		s, _ := snapshot(prev, 300*time.Millisecond)
		enc := json.NewEncoder(os.Stdout)
		enc.SetIndent("", "  ")
		_ = enc.Encode(s)
		return
	}
	if *once {
		s, _ := snapshot(prev, 300*time.Millisecond)
		render(s)
		return
	}

	for {
		var s Stats
		s, prev = snapshot(prev, *interval)
		render(s)
	}
}

// snapshot waits, takes a fresh CPU sample, and collects a full Stats frame.
// It returns the new CPU sample so the caller can reuse it as the next "prev".
func snapshot(prev cpuSample, wait time.Duration) (Stats, cpuSample) {
	time.Sleep(wait)
	cur := readCPUSample()
	return collect(prev, cur), cur
}
```
</details>

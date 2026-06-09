# Example Projects

Small, buildable projects that turn the Linux concepts from the [26-step plan](../README.md)
into something you actually ship. Where the numbered lessons teach commands, these
teach **systems** — a real program, packaged in Docker, deployed to a server.

Same rule as the rest of the course (see [PROGRESS.md](../PROGRESS.md)):
**Claude guides, you build and run it.** Each project is a lesson made of code
blocks and instructions — you create the files, type the commands, and watch it
work. Ask Claude to review what you built or invent extra drills.

## Folder convention

```
example-projects/{project-name}-{level}-{language}
```

- **project-name** — short, lowercase, what the thing is (`gomon`, `logwatch`, …).
- **level** — `beginner` · `intermediate` · `advanced` (roughly how much Linux/infra
  it leans on, not how hard the language is).
- **language** — the implementation language (`go`, `bash`, `python`, …). Omit only
  if it's language-agnostic.

Each project folder holds its own `README.md` — that file *is* the lesson.

## Projects

| Project | Level | Lang | Builds | Touches |
|---|---|---|---|---|
| [gomon](gomon-beginner-go/) | beginner | go | A mini `htop`: a CLI that reads `/proc` and shows live CPU/mem/load/top-processes | `/proc` (steps 04, 11), Docker + Compose (steps 21, 24), deploy to a server |

> More projects get added as you progress. Good next candidates once you've done
> the matching lessons: a log-watcher (steps 09, 20), a tiny web service behind
> nginx (steps 22–24), a backup-and-rotate script (steps 19–20).

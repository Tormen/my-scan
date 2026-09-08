# my-scan

Receives what the scanner pushes, waits until the delivery is actually
complete, hands the OCR to `my-ocr`, and files the result. It can also
pull-scan a device directly.

```
usage: my-scan [OPTIONAL] <MANDATORY> parameters.

  my-scan [OPTIONS] [<FNAME.pdf> [SCANNER_PARAMS...]]   scan (the default)
  my-scan scan     [OPTIONS] [<FNAME.pdf> [SCANNER_PARAMS...]]
  my-scan process  [OPTIONS]                            read filenames on stdin
  my-scan setup    [OPTIONS] [go]                       install the Folder Action
  my-scan status   [OPTIONS]                            what the pipeline is doing
  my-scan cancel   [OPTIONS] [<ID>...]                  documents in flight
  my-scan reset    [OPTIONS]                            stuck machinery (never documents)
  my-scan devices  [OPTIONS]                            find scanners on this LAN
  my-scan stamp-version                                 record the commit in this file
```

## How a scan travels

The scanner does not wait to be asked — it **pushes** a finished document into
a watched folder, over SMB or FTP. macOS Folder Actions notice the new file
and run one Automator step:

```
  scanner --SMB/FTP--> drop folder
                            |  Folder Action fires
                            v
     my-scan process
        1. settle    -- wait until the delivery is really complete
        2. check     -- a light file-type check
        3. name      -- <mtime>___<original>.pdf
        4. my-ocr <the whole batch>        <- blocks until it is our turn
        5. file      -- what my-ocr hands back
```

## What my-scan does and does not own

| | my-scan | my-ocr |
|---|---|---|
| owns | receiving a scan: settle, name, file | OCR: FineReader, its Automator wiring, the mutex |
| Folder Action | the scanner's drop folder | `~/Public/,ocr.in` |
| config | `/LINKS/default/my-scan` | its own |

They meet at one point: my-scan runs `my-ocr <files>`. **my-scan never touches
`,ocr.in`**, and my-ocr knows nothing about scanners.

Exactly one FineReader run can exist at a time — one GUI application, one
Automator wiring — and my-ocr is what guarantees it. So my-scan keeps no lock
and no queue: a second `my-scan process` simply blocks inside `my-ocr` until
its turn.

## settle — why a file is not ready just because it exists

A file appears in the drop folder the moment the transfer *starts*. Picking it
up then yields a truncated PDF. `process` blocks until all three hold:

1. **No writer holds it** — `lsof` says no process in `SETTLE_WRITER_PROCS`
   (`smbd`, `pure-ftpd`, `vsftpd`, `ftpd`, …) has it open for writing. This is
   the check that covers SMB *and* FTP.
2. **Its size has stopped changing** for `SETTLE_STABLE_POLLS` polls running —
   this catches writers `lsof` cannot see (another uid, a remote server).
3. **It parses** — `pdfinfo` for a PDF, `identify` for an image.

None is sufficient alone: between a daemon closing its descriptor and the
bytes being complete there is a real window, and a file can sit at a stable
size while still unreadable.

`status -V` reports which of the three a waiting file is still failing. It
never blocks: it samples every file, sleeps **once**, and re-samples — about a
second in total however many files there are, because size stability cannot be
judged from a single observation.

`lsof` lives in a different place per OS and is searched for
(`SETTLE_LSOF_SEARCH`). If it is missing, my-scan **says so** and carries on
with the other two checks rather than skipping the writer check in silence.

## status

```
CONFIG    /LINKS/default/my-scan (via search)
VERSION   my-scan v2.0 (commit a1b2c3d, build 171071b2c1ae)
SCANNER   EPSON EM-C7100 Series (http://…/eSCL)  not answering (normally off) [2]
DROP      ftp  .../ftp/in/scans      3 waiting
QUEUE     2ocr .../01_scans_2ocr     2 files
QUEUE     fail .../02_FAILED_OCR     0 files
FOLDERACT enabled=true workflows=my-scan.workflow
MY-OCR    idle
OCRAPP    FineReader not running (/Applications/FineReader.app)
RUNNING   <none>
```

Every conclusion carries the value it came from, so a wrong one is visible
without debugging. `MY-OCR` is one line from `my-ocr --status` — is the OCR
busy, the fact you ran `status` for. `-V` adds my-ocr's full detail, the
per-file settle report, the plist keys and the workflow path.

`SCAN_DEVICE_USUALLY_OFF=1` makes a powered-down scanner report *not answering
(normally off)* instead of UNREACHABLE, so the normal state is not a fault.

### `status --watch` — the live view

`my-scan status --watch [SEC]` prints the ordinary status once, then **records
every change as a timestamped line** instead of repainting the screen:

```
  00:43:13  + arriving         scan_0042.pdf
  00:43:16  + queued for OCR   2026-09-07_0031.09___scan_0042.pdf
  00:43:16  - arriving         scan_0042.pdf
  00:43:19  + filed            2026-09-07_0031.09___scan_0042.pdf
```

A `clear`-and-repaint loop destroys the scrollback, which is exactly the
history you are watching for. A move shows as a `-` and a `+` because staging
renames the file; no attempt is made to claim the two are the same document
when the name has changed.

The interval must be shorter than the briefest state you want to see. The
quickest is settle, at `SETTLE_POLL x SETTLE_STABLE_POLLS` (3s by default), so
`WATCH_INTERVAL=1` always catches it; fractions work (`--watch 0.5`).
Transitions that are a single `rename()` have no duration at all and are caught
as an appear/disappear pair, never by polling faster.

my-ocr is asked whether it is busy only while the OCR queue is non-empty, and
the scanner is probed once at the start, not on every cycle. It exits by itself
when nothing is in flight and my-ocr is idle -- so it can be left to run down a
backlog. A watch started against an idle pipeline waits instead of exiting.

### `process --retry` — picking leftovers back up

Nothing ever takes a file out of the OCR queue on its own: `process` stages
from a drop folder and hands the batch straight to my-ocr. So a file left in
`01_scans_2ocr` while my-ocr is idle is **stranded** — `status` says so in
those words rather than showing a number you have to interpret.

```sh
my-scan process --retry              # LIST what is stranded. Changes nothing.
my-scan process --retry --all        # hand it back to my-ocr
my-scan process --retry failed --all # ...and the ones my-ocr could not OCR
```

The two sets are deliberately kept apart. A stranded file was simply never
reached — retrying it almost always works. A file in `02_FAILED_OCR` already
defeated the OCR once, so re-feeding the whole folder blindly just repeats the
failure; you have to ask for it. Retried files are already settled and named,
so they go straight to my-ocr — no second settle, no rename.

### Two `process` runs at once

Claiming files out of a drop folder happens under a lock. Two runs really do
overlap — a backlog drain (`process --all`) while a new scan fires its own
Folder Action — and without it both read the same pending list and stage the
same document under two timestamps: OCRed twice, one copy filed, one left
stranded in the queue.

The lock covers **staging only**. Once a file has been moved out of the drop
folder it is uniquely claimed, so OCR and filing run unlocked — my-ocr's own
mutex is what serialises FineReader. A dead owner's lock is broken
automatically and never silently; a live one is waited for
(`STAGE_LOCK_TIMEOUT`).

### Batches are bounded by PAGES

my-ocr holds the OCR lock for a whole call, so one unbounded call means
everything else — a CLI `my-ocr <file>`, a scan that just arrived — waits for
the entire backlog. `OCR_BATCH_MAX_PAGES` (default 25) caps how much goes into
one call, and the lock is released between chunks.

**Pages, not files.** Ten one-page receipts and ten 200-page contracts are the
same file count and wildly different work, so a file count bounds nothing. A
picture counts as one page, and so does anything whose page count cannot be
read — guessing high would starve a batch, guessing zero would let an unbounded
document slip in.

A document larger than the cap still runs, on its own: never starved, just
alone. `OCR_BATCH_MAX_PAGES=0` restores the single-call behaviour.

`status` reports `<documents> / <pages>` per queue. Counting pages costs a
`pdfinfo` per document (~25ms), so above `PAGE_COUNT_MAX_FILES` (default 40)
the default view reports documents only and `-V` counts the pages.

### Input types

PDF, PNG, JPEG and TIFF. Pictures are fine — the OCR application turns them into a
searchable PDF, and one picture is one page. my-scan's `ACCEPT_FILE_TYPE_REGEX`
and my-ocr's `S_EXPECTED_FILE_TYPE___REGEX` must list the same set: a type only
one of them accepts is a document that settles, stages and then fails. The test
suite compares the two and fails if they drift.

Every type on that list was tried against the real OCR application rather than
assumed -- TIFF was added only after FineReader was shown to take one and
return a searchable PDF. WebP is deliberately absent: macOS `sips` cannot even
write it on these machines, and this FineReader predates WebP by years.

## Checking the document, at both ends

A scan is checked on the way IN and the result is checked on the way OUT,
because in between the original is destroyed: my-ocr replaces it in place and
moves it to the Trash a moment earlier.

**On the way in**, once per delivery after it settles: `qpdf --check`. settle's
own parse test only proves `pdfinfo` can read the catalogue; a file whose page
CONTENT was damaged in transit passes that and fails this. qpdf's exit is
graded and read as such -- 0 clean, 3 warnings only, 2 damage -- because
treating every non-zero as damage would quarantine healthy documents over
cosmetic warnings.

**On the way out**, before the result may replace the original, my-ocr requires
all of: it parses, its page count is **not lower** than the input's, its
Producer names the OCR application, `qpdf --check` passes, and it carries a
text layer. More pages out is fine -- the OCR application splits a sheet it
believes holds several pages.

Deliberately NOT checked: file size (MRC compression makes a much smaller
result the healthy outcome) and image counts (MRC splits one scan image into
mask/foreground/background layers).

If any gate fails it is a clean rollback: the bad result goes to the Trash, the
ORIGINAL is kept and quarantined to `02_FAILED_OCR`, and you are notified with
the reason and where the document now is.

## cancel — documents

```sh
my-scan cancel              # LIST what is in flight. Changes nothing.
my-scan cancel <ID>...      # cancel those documents
my-scan cancel --all        # cancel all of them
```

```
ID  DOCUMENT                                     STATE             SINCE
 1  2026-09-06_1432___scan20260906_143201.pdf    OCRing            3m
 2  2026-09-06_1435___scan20260906_143502.pdf    queued for OCR    40s
 3  scan20260906_143710.pdf                      arriving (SMB)    5s
```

The states are the pipeline as you experience it. Cancelling moves the
document to `02_FAILED_OCR` — **nothing is ever deleted**. A document already
filed in `T_PATH_DONE` is finished work and is not listed.

## reset — machinery

```sh
my-scan reset               # LIST what is not clean. Changes nothing.
my-scan reset --all         # put it back to clean
```

Stray processes, and — delegated to `my-ocr reset` — my-ocr's own machinery:
a mutex whose owner is provably dead, leftovers from runs that are gone, a
stale lockfile. my-scan does not reach into my-ocr's state files; my-ocr owns
that machinery, so it owns the cleanup.

**It never deletes a document.** A document found stranded mid-flight by a
crashed run is *moved* to my-ocr's `RESET_RETRY_TO` (default
`~/Public/,ocr.retry`) and the command to retry it is printed — recoverable,
never lost. That is what makes reset safe to run when you have no idea what is
wrong. Use `cancel` for documents you want to stop.

A **live** run is never interrupted: `my-ocr reset` refuses while one holds the
mutex, because a long OCR is not a crashed one. my-ocr also self-heals — every
run sweeps leftovers from runs that are gone before creating its own, so a
crash cannot poison the next invocation. Staleness is decided by whether the
owning pid still exists, never guessed from age.

Both list on a bare invocation, so the destructive form is never what you get
by accident. `--all` asks first, and with no terminal to ask it refuses unless
`--yes` is given too.

## setup / uninstall

```sh
my-scan setup               # dry run: prints the whole plan, changes nothing
my-scan setup go            # applies it
my-scan install [go]        # a synonym for setup
my-scan uninstall [go]      # the inverse: take the wiring back out
```

For the current GUI user, from the config alone: checks Automator, the OCR
application and every external command; creates the directories the config
names and reports any that exist but are not writable (it never changes an
owner or a mode); writes a **one-step** `my-scan.workflow`, backing up any
existing one, and lints it; attaches it to each `DROP_PATHS` folder through
System Events — which mints the security-scoped bookmarks itself, so the
`NSKeyedArchiver`-encoded preferences plist never has to be written by hand.

It then reads the bindings back and prints what Folder Actions Setup should
show, marking its own. If macOS refuses on Automation permission it names the
setting to change and gives the manual route.

`uninstall` removes **only what setup put in**: its own Folder Action script
and the workflow setup generated (to the Trash, so it stays recoverable).
Another tool's script on the same folder is left alone, and the folder action
itself survives while any script still needs it. It never touches a document, a
queue or a drop folder — and `setup go` puts everything back.

When a folder already carries another script, `setup` **warns rather than
refuses**: a second workflow can be deliberate, but macOS fires every attached
script for each new file, so each document would be processed more than once.
The warning names the script and how to detach it.

## Options

| | |
|---|---|
| `-Q`, `--quiet` | quiet except errors — for cron and LaunchDaemons |
| `-V`, `--verbose` | echo each command and explain it inline as it runs |
| `-D`, `--debug [PATH]` | everything `-V` shows, plus diagnostics |
| `-DD`, `--deepdebug [PATH]` | plus full shell tracing |
| `-L`, `--no-log` | do not write to the configured logfile |
| `--config <FILE>` | use this config, bypassing the search |
| `--create-config [<FILE>]` | print the default config, or write it to FILE |
| `--dry-run` | show what would change; change nothing |
| `--all` / `--yes` | `cancel`/`reset`: act on everything / skip the prompt |
| `--version` | version, commit and build id |

Scan options: `-w/--wait`, `-d/--duplex`, `-r/--resolution <DPI>`, `-g/--grey`.
Settle: `--settle-timeout <SEC>`.

## Configuration

One file, searched in order, first existing wins:

```
$MY_SCAN_CONFIG,  --config <FILE>,  /LINKS/default/my-scan,
~/.my-scan.conf,  /etc/my-scan.conf,  /usr/local/etc/my-scan.conf
```

No behaviour default lives in the script — the only place they exist is the
text `--create-config` emits. The config is validated as it loads: a key that
is missing or not a number is named and refused, because an empty numeric
bound would otherwise turn a wait loop into a spin. After an upgrade:

```sh
my-scan --create-config | diff - /LINKS/default/my-scan
```

| | |
|---|---|
| `DROP_PATHS` / `DROP_LABELS` | watched folders the scanner pushes into |
| `PART1_T_PATH` | where a settled delivery waits for OCR |
| `T_PATH_DONE` / `T_PATH_FAILED` | filed, and failed out |
| `MY_OCR` | the OCR tool. my-scan does the scanning; my-ocr does the OCR |
| `SETTLE_*` | timeout, poll, stability count, writer list, `lsof` search |
| `OCRAPP_*` | which OCR application, for `status` and `setup` |
| `WORKFLOW_*` | what `setup` writes, and the path the workflow calls |
| `SCAN_*` | device, mode, resolution, output folder, viewer |
| `NO_OCR_PATTERNS` | filenames filed without OCR (default `no-ocr-scan*`) |

## Filenames

A delivery is renamed to `<mtime>___<original>.<ext>` — the timestamp comes
from the file's mtime unless the name already starts with one. That is the
name it keeps: nothing rewrites it afterwards.

## scan — generic, not Epson-specific

my-scan does not know about any scanner model. It shells out to a CLI named in
the config; the model lives in `SCAN_DEVICE`, one config string, no code.

`epsonscan2` is not in Homebrew (Epson ships a `.pkg`), and neither is
`sane-airscan`. What is:

```sh
brew install sane-backends      # -> scanimage, plus the epson2 backend
```

so `SCAN_DEVICE="epson2:net:<ip>"` needs nothing vendor-specific. `setup`
warns rather than fails when `scanimage` is absent, since the push pipeline
does not need it.

Only the `scanimage` backend is implemented. An eSCL/AirScan URL or an
`epsonscan2` device string is recognised, and `scan` then says that backend is
not built yet rather than pretending the device string is wrong.

A bare `my-scan` prints the usage — `scan` is the one subcommand that drives
hardware, so it is never what a stray Enter starts.

`SCAN_VIEWER_CMD` opens the finished scan — `open -a Preview` by default,
empty to open nothing.

## Versioning

`--version` identifies the exact bytes: the semantic version, the commit if
stamped, and a hash of the file. Comparing two installs is `--version` on each
and a diff.

```sh
git commit ... && my-scan stamp-version && git push
```

`stamp-version` refuses when another file is dirty or when HEAD is already
pushed, and is idempotent. The stamped sha lags HEAD by one — amending changes
the sha; the build id is authoritative either way.

## The Automator step: which shell

`setup` generates the workflow, and the shell of its *Run Shell Script* step is
`WORKFLOW_SHELL` (default `/bin/zsh`). This is **not** my-scan's own shell -- it
is the one-line glue Automator runs, and **Automator accepts only the shells in
its own list**:

```
/bin/bash /bin/csh /bin/ksh /bin/sh /bin/tcsh /bin/zsh
/usr/bin/perl /usr/bin/ruby /usr/local/bin/python /usr/local/bin/python3
```

`/bin/dash` is not among them, and the failure is silent: the step does
nothing, no error is raised, and nothing appears in any log -- the Folder
Action simply looks dead. `setup` therefore refuses a shell Automator does not
offer, and falls back to `/bin/zsh` when the config predates the setting.

The workflow also records the folder it serves (`folderActionFolderPath`);
without it Automator shows "Folder Action receives files and folders added to:
Choose folder" and nothing arrives. Both halves matter: the folder inside the
workflow, and the binding of that workflow to the same folder.

Chained Folder Actions do **not** deadlock: my-scan's action calls my-ocr,
which drives its own action on `,ocr.in`, and the round trip completes in
about half a minute. `FolderActionsDispatcher` does not serialise globally.

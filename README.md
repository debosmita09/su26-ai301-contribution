# su26-ai301-contribution
# pwndbg Issues 

# Contribution 1: speed up the kernel images download #3005

**Contribution Number:** 1 
**Student:** Debosmita Mallick
**Issue:** https://github.com/pwndbg/pwndbg/issues/3005 
**Status:** Phase I: Completed | Phase II: Completed | Phase III: Completed | Phase IV: Completed

---

## Why I Chose This Issue

I picked issue #3005, "Speed up the kernel images download", from the pwndbg repository because it is a concrete performance problem with a clear and measurable fix. CI is largely unaffected since kernel images are cached after the first run, but first-time local downloads are noticeably slow because every image is downloaded sequentially.

The proposed solution was to parallelize the existing download script without introducing any additional dependencies. Since this only requires modifying an existing shell script, it is a well-scoped issue that still has a meaningful impact on contributor experience.

Before writing any code, I introduced myself on the issue thread: https://github.com/pwndbg/pwndbg/issues/3005 and proposed parallelizing the download script without adding extra dependencies. Rather than assuming how the kernel images were managed, I asked whether the maintainers controlled the image hosting because that would determine whether compressed images could realistically be supported. The maintainers directed me to the existing download script (`tests/library/qemu_system/download-kernel-images.sh`), confirming that it was the correct place to implement the optimization. Later, I continued the discussion about compressed kernel images and dependency management, where the maintainers explained that they preferred avoiding new dependencies and that compressed images would require infrastructure changes outside the scope of this issue. I also connected with the community on Discord to better understand the project`s workflow before beginning implementation.

---

## Understanding the Issue

### Problem Description

The kernel image download script is downloading each image one at a time in a sequential while loop, with each file requiring to be fully completed for the next one to start. This is making the total download time equal to the sum of all individual download times, which is a lot.

### Expected Behavior

All kernel images should download simultaneously in parallel, making the total download time equal to the time taken by the single slowest download file instead of the sum of all individual file download times.

### Current Behavior

Running `./download-kernel-images.sh` downloads each image one at a time. This is evident from seeing the next "Downloading..." line only appearing after the previous file finishes.

### Affected Components

- `tests/library/qemu_system/download-kernel-images.sh` performs all kernel image downloads.
- `tests/library/qemu_system/kernel-tests.sh` invokes the download script whenever cached images are unavailable.

#### Repository Investigation

Before implementing the fix, I investigated how kernel images are downloaded throughout the pwndbg repository instead of immediately modifying the first script I found.

#### Files Investigated: 

- `tests/library/qemu_system/download-kernel-images.sh`: Downloads all kernel images used for kernel testing. This is where the sequential bottleneck exists.
- `tests/library/qemu_system/kernel-tests.sh`: Calls the download script whenever cached images are unavailable.
- `docker-compose.yml`: Used to launch the Ubuntu development environment because setup.sh does not support macOS.
- `docs/contributing/`: Verified contribution workflow, branch policy, and commit conventions.
- `CLAUDE.md`: Reviewed repository-specific development instructions before making changes.

I examined these files to understand where the download process originated, how it was invoked, and whether changes would affect other components of the testing workflow.

#### Maintainer Collaboration

Before implementing the fix, I discussed my proposed solution directly on Issue #3005. My initial proposal was to:

- parallelize the existing download script,
- avoid introducing additional runtime dependencies,
- understand whether compressed kernel images were feasible.

I first asked how the kernel images were hosted because that would determine whether compressed downloads could realistically be supported. The maintainer pointed me directly to `tests/library/qemu_system/download-kernel-images.sh`, confirming that the existing script was the correct implementation location.
After implementing the parallel download solution, I continued the discussion regarding compressed kernel images. The maintainers explained that supporting compressed images would require changes to their image hosting infrastructure and possibly Dockerfiles, but preferred avoiding additional dependencies whenever possible. Based on this feedback, I intentionally kept my implementation focused on parallel downloads while preserving the existing workflow.

#### Acceptance Criteria

Based on the issue description and maintainer feedback, I considered the issue complete only if:

- all kernel image downloads execute concurrently,
- no additional runtime dependencies are introduced,
- the existing workflow remains unchanged,
- failures from individual downloads are still reported correctly,
- the implementation remains limited to the existing download script,
- CI behavior remains unaffected.
---

## Reproduction Process

### Environment Setup

Since pwndbg's `setup.sh` explicitly does not support macOS, I set up my development environment using Docker. I installed Docker Desktop and used the `ubuntu24.04-mount` service from the project's included `docker-compose.yml`, which spins up a Ubuntu 24.04 container with pwndbg pre-installed and the local repo mounted at `/pwndbg`. This meant any file edits I made in VS Code on my Mac were immediately reflected inside the container, and the git history was fully accessible. I also faced a lot of problems setting up the environment along the way like the default branch being `dev` instead of `main`, running git push before creating a personal access token, and running `git add` from the repo root `/pwndbg` inside the container (otherwise the paths were doubled up incorrectly). I also had to configure my git identity inside the container since it started as a blank root environment without any git config.

### Steps to Reproduce

1. Cloned my fork and started the mounted container using:
   `cd ~/pwndbg`, and `docker compose run --rm ubuntu24.04-mount`
2. I navigated to the download script inside the container using: `cd /pwndbg/tests/library/qemu_system`
3. I ran the download script using: `./download-kernel-images.sh`
4. Expected: All "Downloading..." lines appear immediately, showing that the files download in parallel.
5. Observed: Files downloaded one at a time, with each line only appearing after the previous file finishes.

### Reproduction Evidence

- **Commit showing reproduction:** https://github.com/debosmita09/pwndbg/commit/313b3c7dd in the branch: `speed-up-kernel-images-download`, https://github.com/debosmita09/pwndbg/tree/speed-up-kernel-images-download
- **Screenshots/logs:** <img width="1470" height="366" alt="Screenshot 2026-06-14 at 6 18 20 PM" src="https://github.com/user-attachments/assets/a0b396b1-feae-4f7e-9149-395e61210e34" />

- **My findings:** The issue is entirely in the while loop in `download-kernel-images.sh` in lines 28 to 31. The download() function call has no & to background it, so bash waits for each one to return before continuing the loop.

---

## Solution Approach

### Analysis

Root cause: In `tests/library/qemu_system/download-kernel-images.sh` at lines 28–31:

```bash
while read -r hash file; do
    echo "Downloading ${file}..."
    download "${file}"
done < "${OUT_DIR}/hashsums.txt"
```

Each download() call is synchronous. Bash does not proceed to the next loop iteration until wget finishes, with multiple large kernel images.

A secondary issue: the script uses `set -o errexit` at the top. This flag causes the script to exit if a background job (&) fails before wait is called, making it incompatible with the parallel approach without modification.

I searched the repository for existing examples of background jobs (&), PID tracking, and wait usage so that my implementation would remain consistent with the rest of the project. No comparable shell script implementing parallel downloads existed, so I followed standard Bash job-control patterns rather than introducing a custom framework or external dependency.

### Proposed Solution

Background each download() call with &, collect the PIDs, then wait on each one after the loop. Replace `set -o errexit` with `set -o pipefail` to maintain error detection while supporting background jobs.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The download script fetches multiple kernel image files sequentially. First-time contributors experience long waits because files download one at a time instead of concurrently.

**Match:** No parallel download pattern exists elsewhere in the pwndbg codebase to reference. The fix uses standard idiomatic bash: backgrounding jobs with &, storing PIDs in an array, and collecting results with wait.

**Plan:** 
1. Replace `set -o errexit` with `set -o pipefail` in `download-kernel-images.sh` as errexit is incompatible with background jobs
2. Add a `pids=()` array before the while loop
3. Append & to the download `"${file}"` call and push its PID with `pids+=($!)`
4. After the loop, iterate over pids and wait on each; track failures with a failed flag
5. Exit with code 1 if any download failed

**Implement:** https://github.com/debosmita09/pwndbg/tree/speed-up-kernel-images-download, https://github.com/debosmita09/pwndbg/commit/313b3c7dd

**Review:** Read `docs/contributing/` for commit message conventions, PR guideline, and linting requirements. The PR targets the dev branch of pwndbg/pwndbg as required, the project does not use main for development. Keep the change scoped to exactly one file with no unrelated modifications. Run `./lint.sh` inside the container to verify the shell script passes shfmt formatting checks, which is enforced by the pre-push hook installed via setup-dev.sh.

**Evaluate:**
Since this is a shell script that performs external network I/O, there are no automated unit or integration tests to write or run. Verification is observational, i.e after applying the fix, running `./download-kernel-images.sh` shows all "Downloading..." lines appearing before any file completes, confirming parallel execution. Running `time ./download-kernel-images.sh` gives a total wall time of 5 seconds, which is roughly equal to the slowest single download rather than the sum of all downloads. Error handling is also preserved, where if any individual download fails, the script tracks it via the `failed` flag and exits with code 1 with a clear error message.

---

## Testing Strategy

### Unit Tests

This change is in a shell script that performs external network I/O (`wget` downloads). There are no unit-testable functions to isolate. However, I ran a mock-based shell test (`test-parallel-downloads.sh`) to exercise the fix directly without network access. It replaces wget with a fake version that sleeps 1 second per file and records nanosecond-precision start/end timestamps. It asserts that all downloads ran, total wall time is under 2.5s, and timestamps confirm actual overlap between concurrent downloads. This test targets the exact code path changed, which is the backgrounded & loop with wait.

### Integration Tests

The `kernel-tests.sh` script, which calls `download-kernel-images.sh` is the closest integration layer, but it requires live network access and QEMU to run. The CI handles this, and the relevant jobs passed. The two failures, `test_aarch64_store_instructions` and `tests-using-nix` are pre-existing infrastructure issues unrelated to this change, which is documented in the PR comments.

### Manual Testing

- Ran `./download-kernel-images.sh` inside the `ubuntu24.04-mount` Docker container before the fix: files downloaded one at a time, with each "Downloading..." line appearing only after the previous file finished.
- Applied the fix: backgrounded downloads with &, collected PIDs, added wait loop with failure tracking, replaced `set -o errexit` with `set -o pipefail`.
- Ran the script again after the fix: all "Downloading..." lines appeared immediately, confirming parallel execution.
- Ran time `./download-kernel-images.sh`: total wall time was ~5 seconds, roughly equal to the slowest single file rather than the sum of all files.
- Ran `./lint.sh` inside the container: shell script passed shfmt formatting checks enforced by the pre-push hook from setup-dev.sh.
- Error handling verified: the failed flag correctly tracked any download that exits non-zero and exits the script with code 1 with a clear error message.

Beyond the original issue, I also considered several additional scenarios to ensure the implementation remained robust:

- one download fails while the remaining downloads succeed,
- multiple downloads fail simultaneously,
- wait returns processes in a different order than they were launched,
- existing cached images skip downloads entirely,
- future additions to hashsums.txt automatically execute in parallel without requiring code changes.

Tracking every PID individually ensures that each background job is waited on before the script exits, regardless of completion order.

---

## Implementation Notes

### Week 2 Progress

The following notes the things I built and modified:
- Modified `tests/library/qemu_system/download-kernel-images.sh` to parallelize all kernel image downloads.
- Replaced `set -o errexit` with `set -o pipefail — errexit` is incompatible with backgrounded jobs (&) since bash exits if a background process fails before wait is called.
- Added `pids=()` array before the while loop to collect process IDs.
- Appended & to the download `"${file}"` call inside the loop and stored each PID with `pids+=($!)`.
- After the loop, it iterates over pids and waits on each one, where a failed=1 flag tracks any non-zero exit.
- Script exits with code 1 and a clear error message is printed if any individual download fails.
- Added `test-parallel-downloads.sh` with a mock-based test confirming parallel execution without network access.

The challenges I faced included:
- `set -o errexit` silently broke the parallel approach on the first attempt. The script would exit immediately after the first backgrounded job without waiting. Took some investigation to understand why and switch to `set -o pipefail`.
- Had to configure git identity manually inside the Docker container. It was a blank root environment, with no git config by default.
- The project`s default branch is dev not main, so git rebase origin/main failed until I switched to origin/dev.
- Two CI jobs failed with unrelated infrastructure issues: `test_aarch64_store_instructions` (QEMU connection timeout race condition) and `tests-using-nix` (transient Nix cache HTTP 416 error from `cache.nixos.org`). I left comments on the PR explaining that both are pre-existing/infrastructure issues unrelated to this change.

Instead of using a single wait, I waited on each PID individually so that failures from every download could be detected reliably.

I also chose not to introduce tools such as GNU Parallel. While they would simplify concurrent execution, they would also introduce an additional dependency for contributors. Using native Bash job control keeps the implementation lightweight and consistent with the rest of the project.

During discussion, maintainers mentioned that downloading compressed kernel images could provide an additional performance improvement.

Although this idea was interesting, implementing it would require changes to the project`s image hosting infrastructure, decompression workflow, and potentially Dockerfiles. Since Issue #3005 specifically focused on improving download speed, I intentionally limited my implementation to parallelizing downloads while preserving the existing workflow and avoiding unrelated changes.

### Code Changes

- **Files modified:** `tests/library/qemu_system/download-kernel-images.sh`, `tests/library/qemu_system/test-parallel-downloads.sh`
- **Key commits:** https://github.com/debosmita09/pwndbg/commit/313b3c7dd (parallel download fix), https://github.com/debosmita09/pwndbg/commit/1aa252d1b66cc14c12ed6a11513f32f6b95e7c7c (remove .DS_Store), https://github.com/debosmita09/pwndbg/commit/0ad54ca43a21edcd807d809749b5e819c6829605 (mock test)
- **Approach decisions:** I chose wait <pid> per-PID over a bare wait so individual exit codes are captured correctly. Then, I chose `set -o pipefail` over removing error checking entirely to preserve failure detection for pipeline commands elsewhere in the script. I kept the change scoped to exactly one file with no unrelated modifications.

---

## Pull Request

**PR Link:** https://github.com/pwndbg/pwndbg/pull/3972

**PR Description:**  Closing Issue #3005

Speeds up the kernel image download script by parallelizing all wget calls. Previously, `download-kernel-images.sh` downloaded each kernel image sequentially in a while loop, making first-time local setup slow. This change backgrounds each download with &, collects the PIDs, and waits on all of them after the loop, so all images download simultaneously.

Issue #3005 reported that first-time local kernel test setup is unnecessarily slow because images download one at a time. Investigation traced the bottleneck to `tests/library/qemu_system/download-kernel-images.sh` lines 28–31, where download() is called synchronously with no backgrounding. CI is unaffected since images are cached after the first run, but local contributors experience the full sequential wait every time they set up a fresh environment.

Actual description posted:

The `download-kernel-images.sh` script downloaded kernel images sequentially, 
making first-time local setup slow. This change backgrounds each `wget` call 
and waits for all PIDs after the loop so all images download in parallel.

- Replaced `set -o errexit` with `set -o pipefail` (for background job compatibility)
- Collected PIDs after each backgrounded download
- Waits on each PID individually and exits with non-zero if any download fails

+ [x] I read the contributing documentation.
+ [x] I am providing a screenshot of the fixed bug.
<img width="1470" height="375" alt="Screenshot 2026-06-14 at 6 40 59 PM" src="https://github.com/user-attachments/assets/70a6202f-b795-4823-a10a-ff2dc069bf1a" />

**Maintainer Feedback:**
- 06/15/2026: The maintainers supported the parallel download approach and recommended limiting the changes to the existing download script without introducing new dependencies. They also suggested that downloading compressed kernel images could further improve performance but noted that it would require additional changes to the image hosting infrastructure and decompression setup. After benchmarking, they confirmed that the implementation achieved a 2–2.5× speedup, and the only CI failure was an unrelated transient Nix cache issue.
- 06/15/2026: I parallelized the kernel image downloads by running each wget process in the background, tracking all download processes, and waiting for them to complete while preserving proper error handling. I verified that the CI failure was unrelated to my changes, requested a re-run, and the PR was successfully merged after the performance improvements were confirmed. 

**Status:** Merged

---

## Learnings & Reflections

### Technical Skills Gained

- Setting up a Linux dev environment inside Docker on macOS
- Navigating an unfamiliar open source codebase using CLAUDE.md and contributing docs
- Bash parallel job patterns: backgrounding with &, PID arrays, wait with error tracking
- GitHub PAT-based authentication for Git over HTTPS

### Challenges Overcome

- macOS is not a supported platform for pwndbg's setup script, so I had to discover the Docker path by reading README.md and `docker-compose.yml` carefully
- Several layers of git configuration issues inside the container (identity, branch names, auth, working directory) had to be resolved one at a time

### What I'd Do Differently Next Time

- Check `docker-compose.yml` for available services before running any docker compose command
- Configure git identity inside the container immediately after starting it
- Use a PAT credential helper from the start instead of embedding the token in the remote URL

---

## Resources Used

- https://github.com/pwndbg/pwndbg/blob/dev/README.md
- https://github.com/pwndbg/pwndbg/blob/dev/CLAUDE.md
- https://github.com/pwndbg/pwndbg/tree/dev/docs/contributing
- https://github.com/pwndbg/pwndbg/blob/dev/docker-compose.yml
- https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens
- https://www.gnu.org/software/bash/manual/bash.html#Job-Control

  

--------------------------------------------------------------------------------------------



# Contribution 2: Add helpers for reading/writing to physical memory with QEMU #1950

**Contribution Number:** 2
**Student:** Debosmita Mallick
**Issue:** https://github.com/pwndbg/pwndbg/issues/1950
**Status:** Phase I: Completed | Phase II: Completed | Phase III: In-Progress

---

## Why I Chose This Issue

I picked issue #1950, "Add helpers for reading/writing to physical memory with QEMU", from the pwndbg repository because it involves adding a user-facing command that touches multiple layers of the codebase — the command layer, the aglib layer, and the testing infrastructure. After completing PR #3972, I wanted a contribution that required deeper familiarity with how pwndbg structures its internals rather than just modifying a shell script.

The issue is labeled "good first issue" and "help wanted" with no existing PR or assignee, making it an open and well-scoped next step. The proposed fix is self-contained: expose the `Qqemu.PhyMemMode` maintenance packet through a dedicated pwndbg command so that users do not have to send raw GDB maintenance packets manually when debugging kernel memory with QEMU.

Before writing any code, I introduced myself on the issue thread and proposed starting with a `qemu_mem_mode [phys|virt]` command as the simpler option, with `physread/physwrite` as a natural follow-up. I asked the maintainers to confirm the preferred command naming and whether GDB-only scope was acceptable, since `maintenance packet` is a GDB-specific interface with no direct LLDB equivalent.

---

## Understanding the Issue

### Problem Description

When debugging with QEMU, GDB operates in virtual address mode by default. QEMU exposes a maintenance packet interface that allows switching between virtual and physical memory modes, but there is no pwndbg command wrapping this workflow. Users who need to read or write physical memory must manually send raw maintenance packets, which is error-prone and interrupts the debugging workflow.

### Expected Behavior

A dedicated command like `qemu_mem_mode [phys|virt]` should allow users to switch between physical and virtual memory modes cleanly. A `physread/physwrite` command should temporarily switch to physical mode, perform the operation, and restore the previous mode automatically — so the user never has to manage mode state manually.

### Current Behavior

Users must manually run the following GDB commands each time they need to access physical memory:

```bash
maintenance packet qqemu.PhyMemMode    # check current mode
maintenance packet Qqemu.PhyMemMode:1  # switch to physical
maintenance packet Qqemu.PhyMemMode:0  # switch back to virtual
```

There is no pwndbg command wrapping this workflow, and the mode is not restored automatically if something goes wrong mid-session.

### Affected Components

- pwndbg/aglib/kernel/paging.py: already contains switch_to_phymem_mode() at lines 342–384, which saves the current mode, switches to physical, and restores the original mode in a finally block. This is the internal implementation that the new command should expose to users.
- pwndbg/aglib/qemu.py: contains QEMU detection logic (is_qemu(), is_qemu_kernel()) and send_remote() infrastructure used for all maintenance packet communication.
- `pwndbg/commands/`: location for the new command file.
- pwndbg/commands/__init__.py: requires importing the new command module.
- `tests/library/qemu_system/tests/`: location for new kernel tests covering the command.

#### Repository Investigation

Before proposing a solution, I investigated how the `PhyMemMode` packet was already being used in the codebase. I found that `switch_to_phymem_mode()` in pwndbg/aglib/kernel/paging.py already implements the full mode-switching pattern internally for page walks. It saves the old value, switches to physical mode, performs the operation, and restores the previous mode in a `finally` block regardless of success or failure. This means the core logic already exists and the primary work is exposing it through a user-facing command following pwndbg's command architecture described in `CLAUDE.md` and docs/contributing/adding-a-command.md.

#### Files Investigated:

- pwndbg/aglib/kernel/paging.py: existing `switch_to_phymem_mode()` implementation and usage pattern
- pwndbg/aglib/qemu.py: QEMU detection, `send_remote()` calls, and `read_physical_memory()`
- pwndbg/commands/__init__.py: command registration pattern
- docs/contributing/adding-a-command.md: command creation guide
- `CLAUDE.md`: import rules, decorator usage, and architecture overview
- `tests/library/qemu_system/tests/`: existing kernel test patterns to follow

#### Maintainer Collaboration

I introduced myself on issue #1950 and proposed the `qemu_mem_mode [phys|virt]` command as the starting point, with `physread/physwrite` as a follow-up. I asked the maintainers to confirm the preferred command name and whether GDB-only scope was acceptable given that `maintenance packet` has no LLDB equivalent. I am waiting for their response before opening a draft PR.

#### Acceptance Criteria

- `qemu_mem_mode phys` switches GDB to physical memory mode
- `qemu_mem_mode virt` switches back to virtual memory mode
- Running `qemu_mem_mode` with no argument reports the current mode
- The command is gated with `@pwndbg.commands.OnlyWhenQemuKernel` so it only runs in the correct context
- `physread <addr> <size>` reads from a physical address by temporarily switching mode and restoring it after
- `physwrite <addr> <value>` writes to a physical address using the same temporary mode switch pattern
- All existing kernel tests continue to pass
- New tests are added in `tests/library/qemu_system/tests/ covering` the new commands

---
## Reproduction Process

### Environment Setup

Since pwndbg's `setup.sh` explicitly does not support macOS, I set up my development
environment using Docker. I installed Docker Desktop and used the `ubuntu24.04-mount`
service from the project's included `docker-compose.yml`, which spins up a Ubuntu 24.04
container with pwndbg pre-installed and the local repo mounted at `/pwndbg`. This meant
any file edits I made in VS Code on my Mac were immediately reflected inside the container,
and the git history was fully accessible. I faced some friction during setup — the
`docker-compose.yml` lives inside the nested `~/pwndbg/pwndbg/` directory, not the parent
folder, so `docker compose` had to be run from there. The compose service is also named
`ubuntu24.04-mount`, not `pwndbg`, which caused an initial "no such service" error. I also
had to configure my git identity inside the container since it starts as a blank root
environment without any git config.

### Steps to Reproduce

This is a missing-feature issue, so reproduction means demonstrating the gap between what
users must do today and what the issue proposes.

1. Cloned my fork and started the mounted container using:
   `cd ~/pwndbg/pwndbg` and `docker compose run --rm ubuntu24.04-mount`
2. Inside the container, started GDB with pwndbg loaded:
   `gdb --quiet`
3. Inside GDB, loaded pwndbg:
   `python import pwndbg`
4. Attempted to use the proposed command:
   `qemu_mem_mode`
5. **Expected:** A command that switches QEMU between physical and virtual memory modes.
6. **Observed:** `Undefined command: "qemu_mem_mode". Try "help".`
7. Tried the raw manual workaround users must use today:
   `maintenance packet qqemu.PhyMemMode`
8. **Observed:** `packets can only be sent to a remote target` — this error appears because
   no QEMU target is attached, but this raw syntax is what users must memorize and type
   manually in a real kernel debugging session with no automatic mode restoration on failure.

### Reproduction Evidence

- **Working branch:** https://github.com/debosmita09/pwndbg/tree/fix-issue-1950
- **Screenshot:**
  
  <img width="719" height="219" alt="Screenshot 2026-07-12 at 5 31 17 PM" src="https://github.com/user-attachments/assets/949caf06-6366-4671-a8ae-d9082bb11e20" />

- **My findings:** The issue is entirely the absence of a command layer wrapping the
  `PhyMemMode` maintenance packet interface. The internal logic already exists in
  `switch_to_phymem_mode()` at lines 342–384 of `pwndbg/aglib/kernel/paging.py` — it saves
  the current mode, switches to physical, and restores the original in a `finally` block —
  but it is never exposed to users as a GDB command.

---

## Solution Approach

### Analysis

Root cause: No user-facing command exists in `pwndbg/commands/` that wraps the
`qqemu.PhyMemMode` / `Qqemu.PhyMemMode:1/0` maintenance packet workflow. Users who need to
access physical memory during QEMU kernel debugging must manually send raw maintenance
packets, remember the correct packet syntax, and manage mode state themselves — with no
automatic restoration if a session fails mid-operation.

The internal infrastructure already exists: `switch_to_phymem_mode()` in
`pwndbg/aglib/kernel/paging.py` implements the full save/switch/restore pattern, and
`send_remote()` in `pwndbg/aglib/qemu.py` handles all maintenance packet communication.
I confirmed via `git log pwndbg/aglib/kernel/paging.py` that `switch_to_phymem_mode()` was
introduced for internal page-walk use only and was never surfaced as a command. I also
identified the correct decorator for gating QEMU-only commands via
`grep -r "OnlyWhenQemuKernel" pwndbg/commands/`.

I introduced myself on the issue thread and proposed starting with a
`qemu_mem_mode [phys|virt]` command as the simpler option, with `physread/physwrite` as a
follow-up. I asked the maintainers to confirm the preferred command name and whether
GDB-only scope is acceptable since `maintenance packet` has no LLDB equivalent, and am
waiting for their response before opening a draft PR.

### Proposed Solution

Create a new command file in `pwndbg/commands/` that exposes `qemu_mem_mode` as a
user-facing command wrapping the existing `send_remote()` infrastructure, gated with
`@pwndbg.commands.OnlyWhenQemuKernel`. Add `physread` and `physwrite` commands that reuse
`switch_to_phymem_mode()` so mode is always restored automatically.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** When debugging a kernel with QEMU, GDB operates in virtual address mode by
default. QEMU exposes a maintenance packet interface for switching between virtual and
physical memory modes, but pwndbg has no command wrapping this workflow. Users who need to
inspect physical memory must manually send raw maintenance packets and manage mode state
themselves, with no automatic restoration if something goes wrong mid-session.

**Match:** `switch_to_phymem_mode()` in `pwndbg/aglib/kernel/paging.py` (lines 342–384) is
the direct analogue — it already implements the save/switch/restore pattern internally for
page walks. The `@pwndbg.commands.OnlyWhenQemuKernel` decorator found in existing commands
shows the correct pattern for gating commands to the QEMU kernel context.

**Plan:**
1. Create `pwndbg/commands/qemu_mem.py` with a `qemu_mem_mode` command:
   - `qemu_mem_mode phys` → calls `send_remote("Qqemu.PhyMemMode:1")`
   - `qemu_mem_mode virt` → calls `send_remote("Qqemu.PhyMemMode:0")`
   - `qemu_mem_mode` with no argument → reads and prints current mode via `send_remote("qqemu.PhyMemMode")`
   - Gated with `@pwndbg.commands.OnlyWhenQemuKernel`
2. Add `physread <addr> <size>` and `physwrite <addr> <value>` in the same file, reusing
   `switch_to_phymem_mode()` as the context manager so mode is always restored on failure
3. Register the module in `pwndbg/commands/init.py` following the existing import pattern

**Implement:** https://github.com/debosmita09/pwndbg/tree/fix-issue-1950 

**Review:** Read `docs/contributing/index.md` and `CLAUDE.md` for import conventions and
command creation patterns. The PR will target the `dev` branch of pwndbg/pwndbg as
required — the project does not use `main` for development. Run `./lint.sh` inside the
container before pushing. Confirm command naming with maintainers before opening PR since
naming is a maintainer decision.

**Evaluate:**
Manual verification in a live QEMU kernel session:
- `qemu_mem_mode phys` → confirm `maintenance packet qqemu.PhyMemMode` returns `1`
- `qemu_mem_mode virt` → confirm it returns `0`
- Interrupt `physread` mid-operation → confirm mode restores to virtual automatically
- Add automated tests in `tests/library/qemu_system/tests/` following existing kernel test patterns
- Run `./kernel-tests.sh` to confirm no regressions in existing tests

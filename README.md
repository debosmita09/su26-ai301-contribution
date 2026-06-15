# su26-ai301-contribution
pwndbg issue #3005

# Contribution 1: speed up the kernel images download #3005

**Contribution Number:** 1 
**Student:** Debosmita Mallick
**Issue:** https://github.com/pwndbg/pwndbg/issues/3005 
**Status:** Phase I: Completed | Phase II: Completed | Phase III: In-Progress

---

## Why I Chose This Issue
I picked issue #3005 "Speed up the kernel images download" from the pwndbg repo because it's a concrete performance problem with a clear fix. CI is fine since images get cached, but first-time local downloads are slow because everything downloads one at a time.
The fix is to run the download script paralelly, which is just a sript change without any new dependencies. It is a contained but meaningful problem to solve as a first-time contributor. No changes using Python is required unless the CI workflow needs touching.
I left a comment on the issue introducing myself, asked if the parallel downloads approach works for them, and offered to open a draft PR. I also connected with the community via discord to get more clarification on their specific requests.

---

## Understanding the Issue

### Problem Description

The kernel image download script is downloading each image one at a time in a sequential while loop, with each file requiring to be fully completed for the next one to start. This is making the total download time equal to the sum of all individual download times, which is a lot.

### Expected Behavior

All kernel images should download simultaneously in parallel, making the total download time equal to the time taken by the single slowest download file instead of the sum of all individual file download times.

### Current Behavior

Running ./download-kernel-images.sh downloads each image one at a time. This is evident from seeing the next "Downloading..." line only appearing after the previous file finishes.

### Affected Components

tests/library/qemu_system/download-kernel-images.sh is the only file involved with this issue, with kernel-tests.sh calling the download script when no cached images exist.

---

## Reproduction Process

### Environment Setup

Since pwndbg's 'setup.sh' explicitly does not support macOS, I set up my development environment using Docker. I installed Docker Desktop and used the `ubuntu24.04-mount` service from the project's included 'docker-compose.yml', which spins up a Ubuntu 24.04 container with pwndbg pre-installed and the local repo mounted at `/pwndbg`. This meant any file edits I made in VS Code on my Mac were immediately reflected inside the container, and the git history was fully accessible. I also faced a lot of problems setting up the environment along the way like the default branch being 'dev' instead of 'main', running git push before creating a personal access token, and running 'git add' from the repo root '/pwndbg' inside the container (otherwise the paths were doubled up incorrectly). I also had to configure my git identity inside the container since it started as a blank root environment without any git config.

### Steps to Reproduce

1. Cloned my fork and started the mounted container using:
   'cd ~/pwndbg', and 'docker compose run --rm ubuntu24.04-mount'
2. I navigated to the download script inside the container using: cd /pwndbg/tests/library/qemu_system
3. I ran the download script using: ./download-kernel-images.sh
4. Expected: All "Downloading..." lines appear immediately, showing that the files download in parallel.
5. Observed: Files downloaded one at a time, with each line only appearing after the previous file finishes.

### Reproduction Evidence

- **Commit showing reproduction:** https://github.com/debosmita09/pwndbg/commit/313b3c7dd  in the branch: speed-up-kernel-images-download , https://github.com/debosmita09/pwndbg/tree/speed-up-kernel-images-download
- **Screenshots/logs:** <img width="1470" height="366" alt="Screenshot 2026-06-14 at 6 18 20 PM" src="https://github.com/user-attachments/assets/a0b396b1-feae-4f7e-9149-395e61210e34" />

- **My findings:** The issue is entirely in the while loop in download-kernel-images.sh in lines 28 to 31. The download() function call has no & to background it, so bash waits for each one to return before continuing the loop.

---

## Solution Approach

### Analysis

Root cause: In tests/library/qemu_system/download-kernel-images.sh at lines 28–31:

while read -r hash file; do
    echo "Downloading ${file}..."
    download "${file}"   # ← no & = blocks until complete
done < "${OUT_DIR}/hashsums.txt"

Each download() call is synchronous. Bash does not proceed to the next loop iteration until wget finishes, with multiple large kernel images.

A secondary issue: the script uses set -o errexit at the top. This flag causes the script to exit if a background job (&) fails before wait is called, making it incompatible with the parallel approach without modification.

### Proposed Solution

Background each download() call with &, collect the PIDs, then wait on each one after the loop. Replace set -o errexit with set -o pipefail to maintain error detection while supporting background jobs.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The download script fetches multiple kernel image files sequentially. First-time contributors experience long waits because files download one at a time instead of concurrently.

**Match:** No parallel download pattern exists elsewhere in the pwndbg codebase to reference. The fix uses standard idiomatic bash: backgrounding jobs with &, storing PIDs in an array, and collecting results with wait.

**Plan:** 
1. Replace set -o errexit with set -o pipefail in download-kernel-images.sh as errexit is incompatible with background jobs
2. Add a pids=() array before the while loop
3. Append & to the download "${file}" call and push its PID with pids+=($!)
4. After the loop, iterate over pids and wait on each — track failures with a failed flag
5. Exit with code 1 if any download failed

**Implement:** https://github.com/debosmita09/pwndbg/tree/speed-up-kernel-images-download, https://github.com/debosmita09/pwndbg/commit/313b3c7dd

**Review:** Read docs/contributing/ for commit message conventions, PR guideline, sand linting requirements. The PR targets the dev branch of pwndbg/pwndbg as required, the project does not use main for development. Keep the change scoped to exactly one file with no unrelated modifications. Run ./lint.sh inside the container to verify the shell script passes shfmt formatting checks, which is enforced by the pre-push hook installed via setup-dev.sh.

**Evaluate:**
Since this is a shell script that performs external network I/O, there are no automated unit or integration tests to write or run. Verification is observational, i.e after applying the fix, running `./download-kernel-images.sh` shows all "Downloading..." lines appearing before any file completes, confirming parallel execution. Running `time ./download-kernel-images.sh` gives a total wall time of 5 seconds, which is roughly equal to the slowest single download rather than the sum of all downloads. Error handling is also preserved, where if any individual download fails, the script tracks it via the `failed` flag and exits with code 1 with a clear error message.

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** https://github.com/pwndbg/pwndbg/pull/3972

**PR Description:**  Closing Issue #3005

Speeds up the kernel image download script by parallelizing all wget calls. Previously, download-kernel-images.sh downloaded each kernel image sequentially in a while loop, making first-time local setup slow. This change backgrounds each download with &, collects the PIDs, and waits on all of them after the loop, so all images download simultaneously.

Issue #3005 reported that first-time local kernel test setup is unnecessarily slow because images download one at a time. Investigation traced the bottleneck to tests/library/qemu_system/download-kernel-images.sh lines 28–31, where download() is called synchronously with no backgrounding. CI is unaffected since images are cached after the first run, but local contributors experience the full sequential wait every time they set up a fresh environment.

Actual description posted:

The `download-kernel-images.sh` script downloaded kernel images sequentially, 
making first-time local setup slow. This change backgrounds each `wget` call 
and waits for all PIDs after the loop so all images download in parallel.

- Replaced`set -o errexit` with `set -o pipefail` (for background job compatibility)
- Collected PIDs after each backgrounded download
- Waits on each PID individually and exits with non-zero if any download fails

+ [x] I read the contributing documentation.
+ [x] I am providing a screenshot of the fixed bug.
<img width="1470" height="375" alt="Screenshot 2026-06-14 at 6 40 59 PM" src="https://github.com/user-attachments/assets/70a6202f-b795-4823-a10a-ff2dc069bf1a" />

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** Awaiting review

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- https://github.com/pwndbg/pwndbg/blob/dev/README.md
- https://github.com/pwndbg/pwndbg/blob/dev/CLAUDE.md
- https://github.com/pwndbg/pwndbg/tree/dev/docs/contributing
- https://github.com/pwndbg/pwndbg/blob/dev/docker-compose.yml
- https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens
- https://www.gnu.org/software/bash/manual/bash.html#Job-Control

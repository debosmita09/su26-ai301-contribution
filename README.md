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

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

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

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

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

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]

# Instructor notes

## How the per-student uniqueness works

CyberRangeCZ's built-in mechanism for this is **APG (Automatic Problem Generation)**:

1. `variables.yml` declares 4 variables. The `Answers Storage` service generates a fresh value
   for each of them **per sandbox instance**, when the sandbox is allocated to a student.
2. Those same values are exposed as ordinary Ansible variables during provisioning
   (`{{ buf_size }}`, `{{ flag_task2 }}`, etc.) - see `provisioning/roles/victim/tasks/main.yml`.
3. In `training.json`, levels 4-6 (Tasks 2-4) have `"variant_answers": true` and
   `"answer_variable_name"` pointing at the matching variable instead of a static `"answer"`.
   The Training service checks submissions against the Answers Storage value for that specific
   sandbox, not a shared string.

Concretely, per student:

- `buf_size` (type `port`, reused as a generic random integer 60-300) is baked into `stack.c` at
  build time, so the offset from `buffer[]` to the saved return address is different on every
  sandbox. A shared `badfile` almost certainly will not work on someone else's machine.
- `flag_task2` / `flag_task3` / `flag_task4` (type `text`) are random tokens written into
  `/root/flag_task*.txt`, each `chmod 600 root:root`. They cannot be read without an actual root
  shell obtained through the exploit.

## Why the student does NOT get sudo

This was the part most likely to quietly break the whole design, so it's worth spelling out.
The real SEED VM gives the `seed` account full sudo - fine on a personal VM, but on a shared
training platform a student with real sudo could just do:

```
sudo cat /root/flag_task2.txt
```

and skip the exploit entirely. So here:

- `user_access_sudo: False` in `playbook.yml` - the account starts with no sudo at all.
- `/etc/sudoers.d/seed-lab` whitelists exactly **four fixed, literal commands** (see
  `provisioning/roles/victim/templates/seed-lab-sudoers.j2`): toggling
  `kernel.randomize_va_space` between `0`/`2`, and swapping the `/bin/sh` symlink between
  `zsh`/`dash`. None of them can touch file ownership or permission bits, so there's no way to
  turn an arbitrary student-controlled file into a new Set-UID-root binary.
- The vulnerable target (`/usr/local/bin/stack`) is compiled and made Set-UID root by
  **provisioning**, not by the student. If students had a whitelisted `chown`/`chmod`, they
  could point it at a binary of their own choosing and skip the exploit - so that grant was
  deliberately left out, and the target ships pre-built instead.
- Tasks 5 and 6 (StackGuard / non-exec stack) don't need root or Set-UID at all - the canary
  abort and the NX-bit segfault happen identically whether the binary is Set-UID or not, so
  students compile and run those themselves as an ordinary file. No sudo grant needed there.

## Scoring (20 pts)

| Level | Flag | Pts | Why |
|---|---|---|---|
| Turn off countermeasures | static | 1 | Setup, low difficulty |
| Task 1: Run shellcode | static | 2 | Warm-up, no exploit yet |
| **Task 2: Exploit → root** | **APG** | **8** | The core skill; heaviest weight |
| Task 3: Defeat dash | APG | 4 | Builds directly on Task 2, meaningful but smaller lift |
| Task 4: Defeat ASLR | APG | 2 | Kept low-value - see caveat below |
| Task 5: StackGuard observation | static | 2 | Conceptual, no exploit-dependent secret |
| Task 6: Non-exec stack observation | static | 1 | Conceptual, quick |

Static-flag levels are static on purpose, not an oversight: there's no exploit secret to leak
there (the "answer" is a fixed observation - a printed string, a `readlink` result), so per-
sandbox variance wouldn't add anti-cheat value, only friction.

## Known caveats - please check before a live run

1. **IA32 emulation.** The shellcode is 32-bit (`int 0x80`, `%eax`/`%ebx`...), and `stack.c` /
   `exploit.c` are compiled with `-m32`. This requires the `debian-12-x86_64` kernel image used
   by your CyberRangeCZ deployment to support running 32-bit binaries. Most stock kernels do,
   but some hardened/cloud kernel builds disable IA32 compat. **Provision one test sandbox and
   confirm `gcc -m32 -z execstack -o call_shellcode call_shellcode.c && ./call_shellcode` gives a
   shell before releasing this to 30 students.**

2. **Task 4 (ASLR brute force) is not classroom-feasible at full entropy.** Real 32-bit ASLR on
   Linux has ~2^19 possible stack bases - a genuine brute force averages hours, not minutes. The
   level text currently tells students the search space "has been narrowed" via a hint, but
   *the narrowing itself is not yet automated* - right now the hint just says "ask your
   instructor" as a placeholder. Before running this, either:
   - have me add a provisioning step that computes and stores the real (non-ASLR) stack base
     for that sandbox at build time, then reveal only the low ~14 bits as guessable in the paid
     hint, or
   - swap Task 4 for a lower-stakes "write and explain a brute-force script, no live success
     required" grading style, which needs a training.json content edit but no infrastructure
     change.
   I left it at 2 of 20 points precisely because of this uncertainty - happy to firm it up
   either way.

3. **Sequencing gap.** Because `/root/flag_task3.txt` and `flag_task4.txt` are created at
   provisioning time (not on-demand), a very curious student could `ls /root` (if they can get
   any root shell at all, e.g. via Task 2) and see the later filenames before reaching those
   levels in the platform UI. They still can't read the contents without redoing the exploit
   under that task's specific conditions (dash / ASLR), and the platform's level-gating means
   they won't see the *task instructions* naming those files until they get there anyway. This
   mirrors a similar minor gap in the original `library-junior-hacker` example and is not
   considered a real cheating vector, but flagging it for transparency.

4. **BUF_SIZE range (60-300).** Chosen to stay inside the SEED handout's own suggested 0-400
   window while avoiding the extremes. With 30 students and a 240-value range there's a real
   chance of two students landing on the same `BUF_SIZE` by chance (~15% probability of at least
   one collision with ~30 draws from 240 values, birthday-paradox style) - that's fine, since
   the *flag tokens* are still independently random per student regardless of any `BUF_SIZE`
   collision, so a shared badfile between two such students would work but wouldn't let either
   read the other's flag. If you want zero possibility of `BUF_SIZE` collisions, tell me and
   I'll switch it to a deterministic, collision-free function of `global_sandbox_id` instead of
   the `port`-type random generator.

5. **Untested against a real OpenStack image.** I built this from the platform's documentation
   and the `library-junior-hacker` example, but have not been able to provision or run it
   end-to-end (I don't have access to your CyberRangeCZ deployment). Please dry-run it against
   one sandbox before assigning it to the class.

## Setup checklist

1. Push this directory as a Git repository, register it as a Sandbox Definition.
2. Create the Training Definition from `training.json` (or import it directly, if your platform
   version supports JSON import) and confirm the platform reports the `variables.yml` variable
   names match - it will warn you at training-instance creation if they don't.
3. Provision one sandbox, log in as `seed`/`BufferLab123`, and walk through all 6 tasks yourself
   end to end, at whatever `BUF_SIZE` you happen to draw, before assigning it.
4. Adjust `incorrect_answer_limit` / hint penalties in `training.json` to taste.

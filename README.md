# Buffer Overflow Vulnerability Lab (CyberRangeCZ Platform)

A single-host sandbox for [CyberRangeCZ Platform](https://docs.platform.cyberrange.cz/), adapting
SEED Labs' *Buffer Overflow Vulnerability Lab* (W. Du, Syracuse University) into a graded, per-student
exercise. Built to mirror the structure of the `library-junior-hacker` example sandbox
(`topology.yml` + `training.json` + Ansible `provisioning/`).

## Game levels summary

- turn off ASLR / point `/bin/sh` at zsh
- run given shellcode to spawn a shell (warm-up)
- build a working exploit (shellcode + NOP sled + return address) against a Set-UID root binary
- defeat dash's Set-UID privilege-drop countermeasure
- defeat address-space randomization via a (narrowed) brute-force search
- observe StackGuard stopping the same exploit
- observe a non-executable stack stopping the same exploit

## Topology summary

| Host | Image | Flavor |
|---|---|---|
| victim | SEEDUbuntu-16.04-32bit | standard.medium |

Single host + router (router is required plumbing for WAN/user access; the student never interacts
with it directly - same as the SEED lab's single-VM experience).

## Why every sandbox is different

This is an **APG (Automatic Problem Generation)** training - see `variables.yml`. Each sandbox
instance gets:

- its own random `BUF_SIZE`, baked into the vulnerable `stack.c` at provisioning time (so the
  offset a student needs to overwrite the return address is different from everyone else's)
- three independent random flag tokens (`flag_task2`, `flag_task3`, `flag_task4`), written into
  root-only files that are only reachable through an actual root shell obtained via the exploit

See `INSTRUCTOR_NOTES.md` for the full anti-cheat design, its limits, and what to check before
running this with a live class of ~30.

## Scoring

20 points total across 7 graded levels. See `training.json` (`max_score` per level) or
`INSTRUCTOR_NOTES.md` for the breakdown and rationale.

## License

Dual licensing, same approach as `library-junior-hacker`:

* Code (Ansible, templates) - MIT License.
* Game design / level text - CC BY 4.0, adapted from SEED Labs' *Buffer Overflow Vulnerability
  Lab* (Copyright © 2006-2016 Wenliang Du; free to use for non-commercial educational purposes).
  See `ATTRIBUTION.txt`.

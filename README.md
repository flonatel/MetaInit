# MetaInit
Run multiple init systems in parallel

## Introduction
For most of the last 30 years there was exactly one init system for
*nix available: SYSV-init.  There were some changes in syntax and
semantics, but the major functionality is still the same as in the
beginning.

Nowadays there are some init systems available: upstart, nosh, runit,
systemd (to name only a few).

This leads to a major problem:
A user wants to use two programs A and B. A uses initA as it's init
systems, B uses initB as it's init system.  Because (at least at the
moment) it is not possible to run initA and initB in parallel on one
system, it is not possible to use program A and B at the same time.

## Evolution
Because of the complexity of init systems, there will be some
dedicated steps for developing a systems which allows to use more than
one init system in parallel.

### Step 1: metainit and systemd
The first step is to _just_ run metainit and start systemd with
PID!=1.

#### Status
After applying some [patches for
systemd](patches/systemd-1faef9059081b821b7d7a4a1e65013cd8beaaca3.diff)
it was possible to boot the system with systemd not running as PID!=1:

    root@doubleinit:~# ps -elf | grep systemd
    4 S root         1     0  0  80   0 -  1052 -      11:28 ?        00:00:00 /usr/bin/pipexec -l 1 -- [SYSTEMD /lib/systemd/systemd --system ]
    4 S root       137     1  0  80   0 -  7703 -      11:28 ?        00:00:00 /lib/systemd/systemd --system
    4 S root       161   137  0  80   0 -  7120 -      11:28 ?        00:00:00 /lib/systemd/systemd-journald
    4 S root       175   137  0  80   0 -  8736 -      11:28 ?        00:00:00 /lib/systemd/systemd-udevd
    4 S root       341   137  0  80   0 -  7064 -      11:28 ?        00:00:00 /lib/systemd/systemd-logind
    4 S message+   342   137  0  80   0 - 28900 -      11:28 ?        00:00:00 /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation
    4 S root       373   137  0  80   0 -  7282 -      11:33 ?        00:00:00 /lib/systemd/systemd --user
    0 S root       397   376  0  80   0 -  3179 -      12:54 pts/0    00:00:00 grep systemd

But it is currently degraded

    ● doubleinit
        State: degraded
         Jobs: 0 queued
       Failed: 6 units
        Since: Tue 2014-10-28 11:28:20 CET; 1h 23min ago
       CGroup: /
               ├─  1 /usr/bin/pipexec -l 1 -- [SYSTEMD /lib/systemd/systemd --system ]
               ├─137 /lib/systemd/systemd --system

Further investigation is needed to get a stable system.

## Howto
Install [pipexec](https://github.com/flonatel/pipexec).

Copy the `src/metainit` file to `/sbin`. Be sure to make it executable:
`chmod 755 /sbin/metainit`.

In grub, remove the 'quiet' and instead add 'init=/sbin/metainit'.

## Requirements and Design Decisions

### DD: metainit uses pipexec as PID1
[pipexec](https://github.com/flonatel/pipexec) is used as PID1
process.  It is able to start processes and terminate when all
children terminate.



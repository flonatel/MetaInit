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

## Evolution of Development
Because of the complexity of init systems, there will be some
dedicated steps for developing a systems which allows to use more than
one init system in parallel.

### Step 1: metainit and systemd
The first step is to _just_ run metainit and start systemd with
PID!=1.

#### Status
After applying some [patches for
systemd](http://https://github.com/flonatel/systemd-pne1)
it was possible to boot the system with systemd not running as PID1:

    root@doubleinit:~# ps -elf | grep systemd
    4 S root         1     0  0  80   0 -  1053 -      08:10 ?        00:00:00 /usr/bin/pipexec -l 1 -- [SYSTEMD /lib/systemd/systemd --system --basic-system-setup ]
    1 S root       240     1  0  80   0 -  3785 -      08:10 ?        00:00:00 /sbin/cgmanager --daemon -m name=systemd
    4 S root       252     1  0  80   0 -  7584 -      08:10 ?        00:00:00 /lib/systemd/systemd --system --basic-system-setup
    4 S root       272   252  0  80   0 -  7121 -      08:10 ?        00:00:00 /lib/systemd/systemd-journald
    4 S root       278   252  0  80   0 -  8704 -      08:10 ?        00:00:00 /lib/systemd/systemd-udevd
    4 S root       381   252  0  80   0 -  7065 -      08:10 ?        00:00:00 /lib/systemd/systemd-logind
    4 S message+   382   252  0  80   0 - 12517 -      08:10 ?        00:00:00 /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation
    4 S root       409   252  0  80   0 -  6806 -      08:10 ?        00:00:00 /lib/systemd/systemd --user

The following serives are disabled:

    systemctl disable kbd.service
    systemctl disable keyboard-setup.service
    systemctl disable networking.service
    systemctl disable console-setup.service
    systemctl disable selinux-basics.service

The selinux-basic was already started with the metainit script.

System status

    ● doubleinit
        State: running
         Jobs: 0 queued
       Failed: 0 units
        Since: Mon 2014-11-03 08:10:34 CET; 24min ago
       CGroup: /
               ├─  1 /usr/bin/pipexec -l 1 -- [SYSTEMD /lib/systemd/systemd --system --basic-system-setup ]
               ├─240 /sbin/cgmanager --daemon -m name=systemd
               ├─252 /lib/systemd/systemd --system --basic-system-setup
               ├─system.slice
               │ ├─dbus.service

Further investigation is needed to get the disabled system serices up and running.

## Howto
Install [pipexec](https://github.com/flonatel/pipexec).

Copy the `src/metainit` file to `/sbin`. Be sure to make it executable:
`chmod 755 /sbin/metainit`.

In grub, remove the `quiet` and instead add `init=/sbin/metainit`.

## Problems and Open Issues

* Because different init systems expect different setups at system
  init time.  Example: systemd assumes that the cgroup initialization
  is done as PID1.  There is no (easy) way of doing such
  initialization inside another init system.

* Different init systems assume different system services. Examples:
  systemd-logind, syslog vs. journald.

## Requirements and Design Decisions

### DD: metainit uses pipexec as PID1
[pipexec](https://github.com/flonatel/pipexec) is used as PID1
process.  It is able to start processes and terminate when all
children terminate.



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

# Results

during my experiments trying to run more than one init system at a
time I stumbled over many problems (*3).  Meanwhile I'm convinced that
the most problems are results of a non existing or not implemented
concept and design of system services.

Some system services that come to my mind (list might not be
complete): 

o Cron System Service
  examples: cron, anacron, mcron, systemd
o Logging System Service
  examples: syslog, rsyslog, journald
o Resource Management
  example: control groups (cgmanager, systemd)
o Security System Service
  examples: selinux, apparmor
o System Initialization Service
  (One shot things like mounting /sys/fs/selinux file system)
  examples: sysv-init, systemd, upstart, runit, nosh
o System Shutdown Service
  examples: sysv-init, systemd, upstart, runit, nosh
o Application Monitoring Service
  (Always on: start processes, monitor, possible restart)
  examples: systemd, pacemaker
o Configuration Database
  currently implemented as files in file system typically under /etc
  and nowadays under /lib
o Status Database
  currently implemented as files in file system typically under /run

Some services can be easily exchanged; example: syslog, rsyslog,
systemd-journald.  ('Domain specific' API exists for years.)

For others it's huge effort: Exchanging the configuration files under
/etc with postgresql needs touching mostly all packages.
(There is no 'Domain specific' API here.)

Running different services of the same type at the same time will
result in chaos.  (I do not say that this is not possible, but
services must be designed to handle this.) (*1)


The following technical solutions for handling dependencies of
applications to system services came to my mind:


S1: Choose one well defined implementation for every system service
    and support this

    Pros:
    o Only one configuration (setup) needed for each application

    Cons:
    o Dependency to 'one vendor'
    o Possible compatibly problems (hurd, BSD, ...)

    Comment: Maybe this is already true for Debian when choosing
             systemd as 'default' for all system services for Jessie.

S2: Define domain specific API for all system services
    and use it (*2)

    Pros:
    o Only one configuration (setup) needed for each application
    o Well defined API

    Cons:
    o Effort!
    o May lead to 'lowest common denominator' features

    Comment: One part of a possible implementation would be to convert
             a well defined configuration file into the appropriate
             format during installation.

S3: Extract dependencies to system services to extra packages

    Example: Instead of providing startup scripts in the
    openssh-server package, create a package openssh-server-sysvinit
    with contains the configuration for sysv-init and a package
    openssh-server-systemd with configuration for systemd.

    Pros:
    o Independent: no need to touch the program package when using
      another system service.
    o Able to use all the features of one special implementation of a
      system service.

    Cons:
    o Effort!
    o Many new packages.
    o Different packages might need different implementations of one
      specific system service which results in a scenario that not all
      packages can be used at the same time on one system.


Comment: IMHO it's the task of a system administrator to choose the
'right' set of system services for appropriate environment.  I think
(S2) is the cleanest solution - but needs a lot of effort.

Kind regards

Andre


(*1) What could be done is use sysv-init as 'System Initialization
     Service' and systemd as 'Application Monitoring Service'.  But
     using both for both is (currently) not possible.  (Dependencies
     could not handled.)
     Even it is very hard to get the sysv-init to the point to do all
     the system initialization which is needed to run systemd.  The
     needed setup is currently buried deep down in the systemd source
     code. 

(*2) Please note that 'API' should be interpreted as 'well defined
     interface': in this way a configuration file could also be part
     of an API.

(*3) See: https://github.com/flonatel/MetaInit

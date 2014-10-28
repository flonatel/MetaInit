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

### Evolution Step 1: metainit and systemd
The first step is to _just_ run metainit and start systemd with
PID!=1.

## Requirements and Design Decisions

### DD: metainit uses pipexec as PID1
[pipexec](https://github.com/flonatel/pipexec) is used as PID1
process.  It is able to start processes and terminate when all
children terminate.



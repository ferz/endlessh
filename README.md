# Endlessh: an SSH tarpit

Endlessh is an SSH tarpit [that *very* slowly sends an endless, random
SSH banner][np]. It keeps SSH clients locked up for hours or even days
at a time. The purpose is to put your real SSH server on another port
and then let the script kiddies get stuck in this tarpit instead of
bothering a real server.

Since the tarpit is in the banner before any cryptographic exchange
occurs, this program doesn't depend on any cryptographic libraries. It's
a simple, single-threaded, standalone C program. It uses `poll()` to
trap multiple clients at a time.



## Usage

Usage information is printed with `-h`.

```
Usage: endlessh [-vh] [-d MS] [-f CONFIG] [-l LEN] [-m LIMIT] [-p PORT]
  -4        Bind to IPv4 only
  -6        Bind to IPv6 only
  -d INT    Message millisecond delay [10000]
  -f        Set and load config file [/etc/endlessh/config]
  -h        Print this help message and exit
  -l INT    Maximum banner line length (3-255) [32]
  -m INT    Maximum number of clients [4096]
  -p INT    Listening port [2222]
  -v        Print diagnostics to standard output (repeatable)
```

Argument order matters. The configuration file is loaded when the `-f`
argument is processed, so only the options that follow will override the
configuration file.

By default no log messages are produced. The first `-v` enables basic
logging and a second `-v` enables debugging logging (noisy). All log
messages are sent to standard output.

    endlessh -v >endlessh.log 2>endlessh.err

A SIGTERM signal will gracefully shut down the daemon, allowing it to
write a complete, consistent log.

A SIGHUP signal requests a reload of the configuration file (`-f`).

A SIGUSR1 signal will print connections stats to the log.

## Sample Configuration File

The configuration file has similar syntax to OpenSSH.

```
# The port on which to listen for new SSH connections.
Port 2222

# The endless banner is sent one line at a time. This is the delay
# in milliseconds between individual lines.
Delay 10000

# The length of each line is randomized. This controls the maximum
# length of each line. Shorter lines may keep clients on for longer if
# they give up after a certain number of bytes.
MaxLineLength 32

# Maximum number of connections to accept at a time. Connections beyond
# this are not immediately rejected, but will wait in the queue.
MaxClients 4096

# Set the detail level for the log.
#   0 = Quiet
#   1 = Standard, useful log messages
#   2 = Very noisy debugging information
LogLevel 0

# Set the family of the listening socket
#   0 = Use IPv4 Mapped IPv6 (Both v4 and v6, default)
#   4 = Use IPv4 only
#   6 = Use IPv6 only
BindFamily 0
```

## Build issues

Some more esoteric systems require extra configuration when building.

### RHEL 6 / CentOS 6

This system uses a version of glibc older than 2.17 (December 2012), and
`clock_gettime(2)` is still in librt. For these systems you will need to
link against librt:

    make LDLIBS=-lrt

### Solaris / illumos

These systems don't include all the necessary functionality in libc and
the linker requires some extra libraries:

    make CC=gcc LDLIBS='-lnsl -lrt -lsocket'

If you're not using GCC or Clang, also override `CFLAGS` and `LDFLAGS`
to remove GCC-specific options. For example, on Solaris:

    make CFLAGS=-fast LDFLAGS= LDLIBS='-lnsl -lrt -lsocket'

The feature test macros on these systems isn't reliable, so you may also
need to use `-D__EXTENSIONS__` in `CFLAGS`.


### BSD and Blacklistd

Originaly developed for NetBSD, blacklistd is adopted from FreeBSD and OpenBSD, maybe from DragonFlyBSD in next future.

https://www.unitedbsd.com/d/63-how-to-use-blacklistd8-with-npf-as-a-fail2ban-replacement
https://reviews.freebsd.org/D8079

I was investigating about sendmail and blacklistd so I've decided to add blacklistd support to endlessh.

    received 0 from poll()
    received 1 from poll()
    processing type=3 fd=6 remote=::ffff:93.39.143.244:10313 msg=endlessh user uid=0 gid=0
    listening socket: ::ffff:144.76.91.66:22
    look:   target:::ffff:144.76.91.66:22, proto:6, family:28, uid:0, name:=, nfail:*, duration:*
    check:  target:587, proto:6, family:*, uid:*, name:*, nfail:3, duration:86400
    check:  target:25, proto:6, family:*, uid:*, name:*, nfail:3, duration:86400
    check:  target:22, proto:6, family:*, uid:*, name:*, nfail:3, duration:86400
    found:  target:22, proto:6, family:*, uid:*, name:*, nfail:3, duration:86400
    conf_apply: merge:      target:22, proto:6, family:*, uid:*, name:*, nfail:3, duration:86400
    conf_apply: to: target:::ffff:144.76.91.66:22, proto:6, family:28, uid:0, name:=, nfail:*, duration:*
    conf_apply: result:     target:::ffff:144.76.91.66:22, proto:6, family:28, uid:*, name:*, nfail:3, duration:86400
    Applied address ::ffff:93.39.143.244:22
    Applied address ::ffff:93.39.143.244:22
    process: initial db state for ::ffff:93.39.143.244:10313: count=0/3 last=1970/01/01 00:00:00 now=2019/07/27 16:15:03
    run /usr/libexec/blacklistd-helper [control add blacklistd tcp ::ffff:93.39.143.244 128 22 ]
    /usr/libexec/blacklistd-helper: Unsupported packet filter
    add returns (null)
    process: final db state for ::ffff:93.39.143.244:10313: count=3/3 last=2019/07/27 16:15:03 now=2019/07/27 16:15:03


As you can see I've not fixed `/usr/libexec/blacklistd-helper` yet.  I'm not really skilled to instruct pf so I would like if
someone with more pf knowledge will fix it to support endlessh.


[np]: https://nullprogram.com/blog/2019/03/22/

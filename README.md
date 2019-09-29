# Audisp-json

This program is a plugin for Linux Audit user space programs available at <http://people.redhat.com/sgrubb/audit/>.
It uses the audisp multiplexer.

Audisp-json correlates messages coming from the kernel's audit (and through audisp) into a single JSON message that is
sent directly to syslog.
The JSON format used is MozDef message format.

Regular audit log messages and audisp-json error, info messages use syslog.

```
  +-----------+            +------------+
  |           |   Netlink  |            |
  |  kernel   +------------>   auditd   |
  |           |            |            |
  +-----------+            +------+-----+
                                  |                +------------+             +--------------+
                           pipe   |                |            |   Syslog    |              |
                                  |         +------> audisp-json+------------>+    Rsyslog   |
                           +------v-----+   |      |            |             |              |
                           |            |   |      +------------+             +--------------+
                           | audispd    +---+
                           |            |  pipe
                           +---------+--+          +------------+
                                     |             |            |
                                     +-------------> Other      |
                                           pipe    | plugins    |
                                                   +------------+

```
## Building

Required dependencies:
- Audit (2.0+)
- libtool

For package building:
- FPM
- rpmbuild (rpm)

### Build targets:
They're self explanatory.

- make
- make rpm
- make deb
- make install
- make uninstall
- make clean
- make rpm-deps # builds deps for amazonlinux, centos, etc.

Note that packaging targets (like `make rpm`) will package an example rule file, but not use it by default. You can move
it where needed manually.


Example to build for CentOS:
```
docker run --rm -ti -v $(pwd):/build centos:7 /bin/bash -c "yum install -y make && cd /build && make rpm-deps && make
rpm"
```

Or for AmazonLinux:
```
docker run --rm -ti -v $(pwd):/build amazonlinux /bin/bash -c "yum install -y make && cd /build && make rpm-deps && make
rpm"
```

Or for Ubuntu:
```
docker run --rm -ti -v $(pwd):/build ubuntu:14.04 /bin/bash -c "apt-get update && apt-get -y install build-essential && cd /build && make deb-deps && make deb"
```

### Mozilla build targets
We previously used audisp-cef, so we would want to mark that package as obsolete.

- make rpm FPMOPTS="--replaces audisp-cef"
- make deb FPMOPTS="--replaces audisp-cef"

## Deal with auditd quirks, or how to make auditd useable in prod

These examples filter out messages that may clutter your log or/and DOS yourself (high I/O) if auditd goes
down for any reason.

### Example for rsyslog

```
    #Drop native audit messages from the kernel (may happen is auditd dies, and may kill the system otherwise)
    :msg, regex, "type=[0-9]* audit" ~
    #Drop audit sid msg (work-around until RH fixes the kernel - should be fixed in RHEL7 and recent RHEL6)
    :msg, contains, "error converting sid to string" ~
```

### Example for syslog-ng

```
    source s_syslog { unix-dgram("/dev/log"); };
    filter f_not_auditd { not message("type=[0-9]* audit") or not message("error converting sid to string"); };
    log{ source(s_syslog);f ilter(f_not_auditd); destination(d_logserver); };
```
###Example for rsyslog

```
    if $programname == 'audisp-graylog' then {
    *.* @graylog.example.com:5514;RSYSLOG_SyslogProtocol23Format
    stop
    }
```
### Misc other things to do

- It is suggested to bump the audispd queue to adjust for extremely busy systems, for ex. `q_depth=512`.
- You will also probably need to bump the kernel-side buffer and change the rate limit in audit.rules, for ex. `-b 16384
  -r 500`.

## Message handling

Syscalls are interpreted by audisp-json and transformed into a MozDef JSON message.
This means, for example, all execve() and related calls will be aggregated into a message of type EXECVE.

Supported messages are listed in the document messages_format.md

## Configuration file

The audisp-json.conf file has a few options:
- `prepend_msg` Specific a string to prepend all messages with. For example, for Fluentd you might want
  `prepend_msg={"message":`.
- `postpend_msg` Similar to `prepend_msg` but for postpending the messages. To complement the previous example you would
  use `postpend_msg=}`.

## Static compilation tips
If you need to compile in statically compiled libraries, here are the variables to change from the makefile,
using statically compiled as an example.

```
    @@ -48,9 +48,11 @@ else ifeq ($(DEBUG),1)
    else
    CFLAGS  := -fPIE -DPIE -g -O2 -D_REENTRANT -D_GNU_SOURCE -fstack-protector-all -D_FORTIFY_SOURCE=2
    endif
    +CFLAGS := -g -O2 -D_REENTRANT -D_GNU_SOURCE -fstack-protector-all -D_FORTIFY_SOURCE=2

    -LDFLAGS        := -pie -Wl,-z,relro
    -LIBS   := -lauparse -laudit `curl-config --libs`
    +#LDFLAGS       := -pie -Wl,-z,relro -static
    +LDFLAGS := -static -ldl -lz -lrt
    +LIBS   := -lauparse -laudit $(pkg-config --static)
    DEFINES        := -DPROGRAM_VERSION\=${VERSION} ${REORDER_HACKF} ${IGNORE_EMPTY_EXECVE_COMMANDF}

    GCC            := gcc
```

NOTE: Any library you enable will need to be available as a static library as well.

# Logcat to UDP

Fork of [Chemik/logcatudp](https://github.com/Chemik/logcatudp) with fixes for
Android 8.1+ (Oreo) foreground service requirements and reliability improvements.

Background service collects logcat output and sends it via UDP to a syslog server
or custom receiver. Useful for monitoring Android devices remotely - even when not
connected via ADB.

Tested on:
- Android 8.1 (Android Things, Lenovo 10" displays)
- Android 10 (Lenovo Tab M10 FHD Plus, rooted with Magisk)
- Android 11 (PHH-su, 7" display)

Pre-built APKs are available on the [Releases page](https://github.com/markuslaube/logcatudp/releases).

## Changes from upstream

- Renamed package to `bi.lau.android.logcatudp` (avoids conflicts with upstream)
- `startForegroundService()` for Android 8+ compatibility
- Retry logic on `ENETUNREACH` (network not yet ready at boot)
- 50ms sleep when logcat buffer empty (fixes 100% CPU busy-loop)
- Signed builds via GitHub Actions with fixed keystore
- Auto-versioning: `versionCode = GITHUB_RUN_NUMBER`, `versionName = 1.<run_number>`

## Granting READ_LOGS permission

LogcatUDP requires `READ_LOGS` to collect system logs. On most Android devices this
can be granted via GUI (Settings > Apps > LogcatUDP > Permissions).

On devices without a standard permission UI (e.g. Android Things), grant it manually
via ADB with root access:

```
adb root
adb shell pm grant bi.lau.android.logcatudp android.permission.READ_LOGS
```

If `adb root` is not available (production firmware), use `su`:

```
adb shell su -c "pm grant bi.lau.android.logcatudp android.permission.READ_LOGS"
```

After granting, start the service:

```
adb shell am start-foreground-service -n bi.lau.android.logcatudp/.LogcatUdpService
```

The service auto-starts on boot via `BOOT_COMPLETED` broadcast receiver.

## HOWTO Recieve Logs

### Introduction

So you have installed application on your android device. That's great :)

But how you receive and/or save sended logs on computer?

Here are some ways how to do this:


#### 1. Read log direct from UDP port

To listen logs use listener.py script located in test directory

Example:
```
$ ./listener.py 192.168.1.10 10009
```

To save this output to file use standard linux/unix commands like output redirection or [tee](http://unixhelp.ed.ac.uk/CGI/man-cgi?tee).


#### 2. Default syslog server

If you use some kind of syslog server (rsyslog, SyslogNG), you can setup LogcatUDP to send logs to this server.

**Rsyslog** can be configured to listen remote logs on UDP port. Default port is 514. To enable listening uncomment this two lines in `/etc/rsyslog.conf`:
```
$ModLoad imudp
$UDPServerRun 514
```
You can of course change the number of port. With this configuration syslog writes log to number of log files (/var/log/syslog, /var/log/messages, ...). If you want to filter log lines from android and write them only to one special file, you must set up syslog rules. Create file in `/etc/rsyslog.d/` with name that is in front of other files in this directory. For example if the first file is `20-uwf.conf`, then you can name it `10-logcat.conf`. Write this rule to it:
```
#### process remote messages
if $fromhost-ip startswith '192.168.2.' then /var/log/android
& ~
# only local messages past this point
```
The third line (`"& ~"`) is important. This is discard message after write to file.

# ghostlock_5.4_xperia

Temporary root for Sony Xperia 5.4 kernels, based on GhostLock
(CVE-2026-43499). Includes a persistent `su` daemon.

## Supported devices

| Kernel | Devices |
| --- | --- |
| `5.4.210-qgki-g55a0c74ba38a` | Sony Xperia 5 III (SOG05) |
| `5.4.295-qgki-g1da550e1d37c` | Sony Xperia 10 V (SO-52D) |

## Build and run

```sh
make -C exploit list-projects
make -C exploit PROJECT=sog05-5.4.210 NDK_ROOT="$NDK" -j2 bin

adb push exploit/build/sog05-5.4.210/bin/exploit_static /data/local/tmp/gl
adb shell 'chmod 755 /data/local/tmp/gl; /data/local/tmp/gl'
```

Reboot clears root. A failed attempt may reboot the device; rerun.

## su

The exploit creates `/data/local/tmp/su` and starts a root daemon once the
anchor has root and SELinux is permissive:

```sh
/data/local/tmp/su
/data/local/tmp/su -c id
```

See [README_origin.md](README_origin.md) for the original project notes and
[LICENSE](LICENSE) for the terms.

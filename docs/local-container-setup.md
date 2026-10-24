# Running camofox in a locked-down container (no root, no user namespaces)

Notes from getting the Firefox/Camoufox engine running on a hardened container:
uid without sudo, no package manager writes, unprivileged user namespaces blocked,
no `libgtk-3.so.0` in the base image.

## 1. GTK3 without root

`libmozgtk.so` needs GTK3, which the image does not ship. Pull the debs on any
machine, unpack them user-space, and point `LD_LIBRARY_PATH` at the result:

```bash
mkdir -p ~/.local/gtkroot && cd /tmp
# debs are kept afterwards in ~/.cache/pkgcache/apt/archives/
apt-get download libgtk-3-0t64 libgtk-3-common libepoxy0 libcloudproviders0 \
  libcolord2 libxinerama1 libapparmor1 libdconf1 adwaita-icon-theme \
  hicolor-icon-theme gtk-update-icon-cache
for d in *.deb; do dpkg -x "$d" ~/.local/gtkroot; done
```

Then in `~/.bashrc` (arm64 example):

```bash
export LD_LIBRARY_PATH="$HOME/.local/gtkroot/usr/lib/aarch64-linux-gnu:$HOME/.local/gtkroot/lib/aarch64-linux-gnu${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
```

Symptom when missing:

```
XPCOMGlueLoad error for file .../libmozgtk.so:
libgtk-3.so.0: cannot open shared object file: No such file or directory
Couldn't load XPCOM.
```

## 2. Firefox sandboxes

The container blocks `clone()` for user namespaces, so the sandbox cannot start:

```
Sandbox: CanCreateUserNamespace() clone() failure: EPERM
```

```bash
export MOZ_DISABLE_CONTENT_SANDBOX=1
export MOZ_DISABLE_GMP_SANDBOX=1
```

## 3. Start through a login shell

Both exports live in `~/.bashrc`, which non-interactive shells skip. Launch as:

```bash
bash -lic 'set +m; exec npm start'
```

`/health` returning 503 means the browser subprocess failed to launch — check the
two items above first. Healthy:

```bash
curl -s localhost:9377/health
# {"ok":true,"engine":"camoufox","browserConnected":true,"browserRunning":true,...}
```

## 4. node_modules as a symlink (shared cache)

Node resolves package imports through the importing file's **realpath**. If
`node_modules` is a symlink into a shared cache directory that is *not* named
`node_modules`, sibling packages are no longer reachable from nested packages:

```
Error [ERR_MODULE_NOT_FOUND]: Cannot find package 'playwright-core'
  imported from /path/to/pkgcache/camofox-node_modules/camoufox-js/dist/server.js
```

because resolution looks for `<parent>/node_modules/...` and the cache directory
is called something else. Two fixes:

- keep `node_modules` a real directory (simplest), or
- add the self-referential symlink inside the cache directory:

```bash
ln -sfn . /path/to/pkgcache/camofox-node_modules/node_modules
```

## 5. Optional: telemetry

Anonymized crash/hang reports go to the project's Cloudflare Worker endpoint.
Disable with `CAMOFOX_CRASH_REPORT_ENABLED=false`.
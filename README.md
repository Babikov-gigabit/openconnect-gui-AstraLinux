# OpenConnect GUI — Astra Linux build (Debian 12)

*Русская версия: [README.ru.md](README.ru.md)*

This repository is a fork of the official [OpenConnect GUI](https://gitlab.com/openconnect/openconnect-gui)
project, adapted so it builds and packages cleanly as a native `.deb` on
**Astra Linux 1.8** (Debian 12 "bookworm" based). All application code is
unmodified upstream OpenConnect GUI; the changes here are limited to the
Debian packaging (`debian/`) needed to make the build succeed on Astra Linux.

## What was fixed and why

Building the existing `debian/` packaging unmodified on Astra Linux 1.8
failed at the `cmake` configure step with:

```
Could not find a package configuration file provided by "Qt6" ...
```

even though `qt6-base-dev` was installed and `Qt6Config.cmake` was present at
`/usr/lib/x86_64-linux-gnu/cmake/Qt6/Qt6Config.cmake`. The cause is that this
CMake build does not resolve `CMAKE_LIBRARY_ARCHITECTURE` on this system, so
it never looks under the multiarch (`x86_64-linux-gnu`) library directory on
its own.

**Fix:** `debian/rules` now passes an explicit
`-DCMAKE_PREFIX_PATH=/usr/lib/x86_64-linux-gnu/cmake` to `cmake`, which lets
it locate `Qt6Config.cmake` and, transitively, `OpenGL`/`GnuTLS` and the
other CMake-config-based dependencies.

No other files were changed. The resulting binary is the stock upstream
OpenConnect GUI 1.6.2.

### A note on Astra Linux's closed software environment

Astra Linux 1.8 runs the `digsig_verif`/`parsec` kernel modules (Astra's
"Замкнутая программная среда" / closed software environment). Under this
policy, an unprivileged user cannot `chmod +x` a newly created file — which
`dpkg-buildpackage` needs to do (e.g. to mark `debian/rules` executable, and
during `dh_fixperms`). **Building must be done as root** (e.g. via `sudo`).
Compiling and linking themselves are unaffected; only permission changes on
freshly created files are restricted for non-root users.

If your working tree lives on a VirtualBox shared folder (`vboxsf`), be
aware that such mounts commonly report every file with a single fixed mode
(e.g. `0770`), which corrupts what `git`/`cp` see as "executable" files when
copied. This repo's build instructions below copy the source tree to a
regular filesystem before building to avoid that.

## Building the package

Install build dependencies (available from Astra Linux's `main` and
`extended` repositories once `apt-get update` has fetched current package
lists):

```bash
sudo apt-get update
sudo apt-get install -y devscripts build-essential debhelper cmake pkg-config \
    qt6-base-dev qt6-base-dev-tools qt6-scxml-dev \
    libgnutls28-dev libopenconnect-dev libspdlog-dev libxml2-dev
```

Build (as root, for the reason explained above; run from a plain filesystem,
not a `vboxsf`/network share):

```bash
sudo dpkg-buildpackage -us -uc -b
```

This produces `openconnect-gui_<version>_amd64.deb` (plus a `-dbgsym` debug
package) in the parent directory.

## Installing

```bash
sudo apt install ./openconnect-gui_1.6.2-1_amd64.deb
```

`apt` resolves `openconnect`, `vpnc-scripts` and the Qt6/GnuTLS/spdlog
runtime libraries directly from Astra Linux's own repositories — no manual
dependency wrangling needed, as long as `apt-get update` has been run
recently.

For machines without access to Astra's package repositories, see
[Releases](../../releases) for an offline bundle containing this `.deb`
together with all of its dependency `.deb` files and an `install.sh` script.

## Upstream project

Everything else in this repository — the application itself, its features,
supported platforms, and development docs — is unchanged from upstream. See
the original project for details:

- [OpenConnect VPN GUI web site](https://gui.openconnect-vpn.net/)
- [Upstream source (GitLab)](https://gitlab.com/openconnect/openconnect-gui)
- [Compilation docs](docs/dev.md) · [QtCreator setup](docs/dev_QtCreator.md)

## License

Licensed under the [GNU General Public License v2](LICENSE.txt), same as
upstream. See [debian/copyright](debian/copyright) for full per-file
copyright attribution. This fork adds no additional copyright claims beyond
the `debian/rules` packaging change noted above.

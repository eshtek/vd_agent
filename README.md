<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/hexos-logo-dark.svg">
    <img src="assets/hexos-logo.svg" alt="HexOS" width="234">
  </picture>
</p>

# SPICE guest agent for Windows, HexOS build

This is the Windows SPICE guest agent (`vdservice.exe` and `vdagent.exe`) that
[HexOS](https://hexos.com) installs in Windows virtual machines. It runs inside
the guest and gives the SPICE console its mouse, its clipboard, file copy into
the VM, and a desktop that resizes to fit the viewer's window.

The agent most Windows VMs run comes from the VirtIO guest tools, and that is
still 0.10.0 from April 2019. Upstream has fixes on its master branch that were
never released, and the freedesktop.org project is no longer maintained. This
repository builds the current code with the patches HexOS needs and publishes
an installer that upgrades the old agent in place.

## What's different from the 0.10.0 agent

- The mouse keeps working next to a passed-through GPU. With a GPU in the VM
  that has a monitor or a dummy plug attached (or with a virtual display
  driver), the old agent dropped every mouse event: the pointer froze in the
  console while the keyboard and Remote Desktop kept working. Upstream fixed
  the case where the GPU has nothing plugged in. This build also handles a GPU
  with a monitor by mapping the mouse across the whole desktop, and it leaves
  the display layout alone so it never moves or resets a real monitor.
- Files copied into the VM land in Downloads, and the agent opens the folder,
  as the Linux agent does. The old agent saved them to the Desktop.
- File copy tells the client when nobody is signed in to Windows, and it
  accepts long file names in non-Latin scripts.
- Side and extra mouse buttons (back and forward) work.
- Clipboard images go through the Windows Imaging Component instead of a
  bundled libpng, with size checks on what the client sends.
- Fixes for crashes and memory errors in display settings, the message queue
  and capability negotiation.
- A newer MSI replaces the installed agent, including the one the VirtIO guest
  tools put there, and the guest tools installer then skips its own older copy
  instead of failing.

The full list is in [CHANGELOG.md](CHANGELOG.md).

## Install

Download `spice-vdagent-x64-<version>.msi` from
[Releases](https://github.com/eshtek/vd_agent/releases) and run it in the VM,
or install it silently from an administrator prompt:

```powershell
msiexec /i spice-vdagent-x64-<version>.msi /qn /norestart
```

It installs the `spice-agent` service (display name "Spice agent") to
`C:\Program Files\SPICE agent\bin`. If the VirtIO guest tools already installed
an agent, this one replaces it. The installers are not code-signed yet, so
Windows may warn that the publisher is unknown.

HexOS installs this agent for you when it sets up a Windows VM from the
catalog. For a VM you installed yourself, install the VirtIO guest tools first
(they bring the virtio-serial driver the agent talks over), then this MSI.

To go back to the stock agent, uninstall "Spice agent" from Installed apps and
run the VirtIO guest tools installer again.

## Troubleshooting

The agent logs to `C:\Windows\Temp\vdagent.log`. Each time the display setup
changes it writes a line like this:

```
consistent_displays::#qxls 1 #others 1
```

`#others` counts displays on adapters other than QXL that have a monitor. When
both numbers are above zero, the 0.10.0 agent stops handling the mouse and logs
`No QXL devices!` at startup. This build keeps going and logs the size of the
whole desktop it maps the mouse onto:

```
fit_virtual_desktop::virtual desktop (0,0) 3200x1080
```

## Building

GitHub Actions builds every push and pull request to `hexos`
([.github/workflows/build.yml](.github/workflows/build.yml)): an MSYS2 UCRT64
build that runs the tests and packages the MSI, an MSVC build, and a Fedora
MinGW cross build. Pushing a `v*` tag publishes a release with the MSI, the
source tarball and a SHA-256 file for each.

To build on Linux or macOS, any Fedora container works:

```sh
git clone --recursive https://github.com/eshtek/vd_agent.git
cd vd_agent
docker run --rm -v "$PWD":/src -w /src fedora:42 bash -c '
  dnf install -y git libtool make autoconf autoconf-archive automake \
    mingw64-gcc-c++ mingw64-winpthreads-static msitools
  git config --global --add safe.directory /src
  autoreconf -if && mingw64-configure && mingw64-make && mingw64-make msi'
```

That leaves `vdagent.exe`, `vdservice.exe` and
`spice-vdagent-x64-<version>.msi` in the checkout. To build on Windows, use an
MSYS2 UCRT64 shell:

```sh
bash msys2/install.sh
autoreconf -i
bash msys2/build.sh builducrt64
bash msys2/package.sh builducrt64
```

The version comes from `git describe`, so build from a clone with its tags.

## Branches and releases

- `hexos` is the default branch and what releases are cut from.
- `master` mirrors [upstream](https://gitlab.freedesktop.org/spice/win32/vd_agent)
  unchanged.

`hexos` is upstream master plus the packaging and hardening work from
[nefarius/vd_agent](https://github.com/nefarius/vd_agent), plus HexOS's own
patches. Releases keep the maintained fork's minor version and count HexOS
releases in the patch number: v0.13.1 is the first HexOS release on top of 0.13.0.
The MSI keeps upstream's UpgradeCode, so each release upgrades any earlier
build, ours or anyone else's.

To release, add the changes to [CHANGELOG.md](CHANGELOG.md), then tag and push:

```sh
git tag -a v0.13.2 -m "vdagent-win 0.13.2"
git push origin v0.13.2
```

## License

GPL-2.0-or-later; see [COPYING](COPYING). The agent is the work of Red Hat and
the SPICE contributors. Benjamin Höglinger-Stelzer
([nefarius](https://github.com/nefarius)) wrote the multi-GPU mouse fix and
keeps the maintained fork this build is based on. SPICE is not a HexOS
trademark, and this build is not affiliated with the SPICE project.

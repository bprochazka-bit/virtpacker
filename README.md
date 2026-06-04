# virtpacker

Export and import libvirt virtual machines as portable, single-directory bundles.

virtpacker takes a libvirt domain and packs everything needed to recreate it on
another host into one directory: the domain XML with host paths templated out,
the disk images, the UEFI nvram variable store, and the swtpm TPM state. You
copy that directory to another machine, run the import, and the VM is defined
and ready to start. The model is deliberately close to a VMware VM folder, where
the config references its disks and firmware state by a relative, self-contained
layout.

There is no single-click export in stock libvirt or virt-manager. A libvirt VM
is really a domain XML plus a set of host-side files that the XML points at, and
some of those files (swtpm state in particular) are not referenced in the XML at
all. virtpacker handles the full set so a move does not silently drop boot
entries, enrolled Secure Boot keys, or TPM-sealed data.

## What it handles

- **Disks.** Copied into the bundle. By default each disk is flattened into a
  standalone qcow2 with `qemu-img convert`, which collapses backing chains and
  produces a self-contained image. `--no-flatten` copies the disk as-is and
  refuses if it detects a backing chain, rather than producing a broken bundle.
  Copies are sparse-aware (via `cp --sparse=always --reflink=auto`), so holes in
  a sparse image are preserved instead of being written out as real zeroes.
- **nvram.** The UEFI variable store is copied into the bundle and its path is
  templated. This file holds the boot order and any enrolled Secure Boot keys.
- **UEFI firmware (loader).** Not copied. The read-only OVMF or edk2 firmware is
  a host package that receives security updates and lives at different paths on
  different distros. On import the domain is switched to firmware autoselection
  (`<os firmware='efi'>`) so each host supplies its own firmware. When the source
  used Secure Boot, the bundle preserves that by emitting `<loader secure='yes'/>`
  and relying on the carried nvram for the enrolled keys.
- **swtpm TPM state.** This is the awkward one, because the state location is not
  always in the XML. virtpacker resolves it in order of authority:
  1. an explicit `<backend type='emulator'><source .../>` path in the domain XML
     (libvirt 10.8.0 and later), which may be a single file or a directory;
  2. otherwise the UUID-derived default, conventionally
     `/var/lib/libvirt/swtpm/<uuid>/`.
  The explicit path is read from the XML rather than assumed, and is templated
  like any other path. The UUID-derived directory is placed back at its UUID path
  on import, since it is not referenced in the XML.
- **SELinux contexts.** On import, files land in the standard libvirt directories
  and are relabeled with `restorecon`. If you direct output to a non-standard
  directory, virtpacker adds a `virt_image_t` fcontext rule with `semanage` first,
  then relabels.

## Requirements

The package depends on `virsh` (from `libvirt-clients`) and `qemu-img` (from
`qemu-utils`). It is pure Python standard library otherwise.

For the imported VMs to actually run, the destination host needs the firmware and
TPM backends, which is why the package recommends `ovmf`, `swtpm`, and
`swtpm-tools`. SELinux relabeling is optional on Debian (which defaults to
AppArmor), so `policycoreutils` and `policycoreutils-python-utils` are suggested
rather than required; without them the relabel step is skipped with a warning.

Export of nvram and swtpm state and all import operations require root, because
they read from and write to `/var/lib/libvirt`.

## Installing

### From the Debian package

```
sudo apt install ./virtpacker_0.3.3_all.deb
```

This installs `/usr/bin/virtpacker` and its man page. See "Building the package"
below to produce the .deb.

### Running directly

The tool is a single script with no third-party Python dependencies, so you can
also just run it in place:

```
sudo ./virtpacker export winvm -o /tank/bundles
```

## Usage

List the domains libvirt knows about, with each domain's state and disk usage
(size on disk versus apparent size):

```
virtpacker list
```

The "On disk" column is the actual allocation (it honors sparseness), while
"Apparent" is the nominal file length; the gap between them is the space a
sparse image is saving.

Export one domain, or all of them:

```
sudo virtpacker export winvm -o /tank/bundles
sudo virtpacker export --all -o /tank/bundles
```

Import a bundle on another host, optionally starting it:

```
sudo virtpacker import /tank/bundles/winvm
sudo virtpacker import /tank/bundles/winvm --start
```

Both export and import run a free-space preflight first: they estimate the
on-disk bytes (sparse-aware) and abort early if the destination filesystem is
short, rather than failing partway through a large copy. Pass `--no-space-check`
to skip it.

Disk copies are atomic (written to a temp file and renamed into place), so an
interrupted import never leaves a truncated image. An existing disk image at the
destination is never overwritten: if the name is already taken, the incoming disk
is copied under a unique name (a short random token is appended) and the domain
XML is updated to point at it. To instead reuse an already-present image — for
example to resume an import that copied the disks but failed at the define
step — pass `--no-overwrite-disks`:

```
sudo virtpacker import /tank/bundles/winvm --no-overwrite-disks
```

On a host that shares a network segment with the source, regenerate the MAC to
avoid a collision:

```
sudo virtpacker import /tank/bundles/winvm --new-mac
```

### Export options

- `-o, --outdir DIR` where to write bundles (default: current directory)
- `--all` export every defined domain
- `--no-flatten` copy disks as-is, refuse on a backing chain
- `--overwrite` replace an existing bundle directory
- `--no-space-check` skip the destination free-space preflight

### Import options

- `--images-dir DIR` disk image destination (default `/var/lib/libvirt/images`)
- `--nvram-dir DIR` nvram destination (default `/var/lib/libvirt/qemu/nvram`)
- `--tpm-dir DIR` destination for an explicit-source swtpm file
  (default `/var/lib/libvirt/swtpm`); UUID-derived TPM state always goes to its
  UUID path
- `--owner USER[:GROUP]` ownership for installed files (auto-detected otherwise)
- `--new-mac` strip fixed MACs so libvirt regenerates them
- `--machine MACHINE` set the machine type (e.g. `q35`); by default a versioned
  type is aliased so the host picks a supported version
- `--keep-machine` keep the bundle's exact machine type instead of aliasing it
- `--no-overwrite-disks` reuse a disk whose destination already exists instead
  of importing a fresh copy under a unique name (e.g. to resume a failed import)
- `--start` start the domain after defining it
- `--no-space-check` skip the destination free-space preflight

## Bundle layout

```
winvm/
  manifest.json         name, uuid, secure_boot flag, disk/nvram/tpm inventory
  domain.xml.template   domain XML with paths replaced by @@bundle/relative@@
  disks/
    winvm-vda.qcow2
  nvram/
    winvm_VARS.fd
  swtpm/                directory, or a single file for an explicit source
```

Paths in the template are written as `@@disks/...@@`, `@@nvram/...@@`, and
`@@swtpm/...@@`. On import each placeholder is resolved to the chosen destination,
the file is copied there, ownership and SELinux labels are applied, and the
placeholder is replaced with the absolute destination path before the domain is
defined.

## Limitations and things to know

- **Flattening drops snapshots.** The default flatten collapses backing chains
  and internal snapshots into a single image. This is usually what you want for a
  portable bundle, but export a snapshot base with that in mind. Use
  `--no-flatten` when you need to keep the disk format intact (and flatten or
  rebase any backing chain yourself first).
- **System-mode only.** The tool assumes `qemu:///system`. Session-mode libvirt
  (`qemu:///session`) uses different base directories and is not handled.
- **localstatedir assumption.** For the UUID-derived swtpm fallback, the base
  `/var/lib/libvirt/swtpm` is assumed. This is correct on every mainstream distro
  but is set at libvirt build time, so a libvirt built with a custom prefix would
  differ. An explicit `<source>` in the XML sidesteps this entirely.
- **Ownership varies by distro.** The qemu user is auto-detected (`libvirt-qemu`
  on Debian, `qemu` on RHEL). Override with `--owner` if your swtpm files need a
  different owner.
- **CPU model.** If the source domain uses `host-passthrough` and the destination
  CPU differs, the guest may not boot. Switch the imported domain to `host-model`
  or a named model in that case.
- **Machine type.** The source records an exact versioned machine type (e.g.
  `pc-q35-10.0`). An older QEMU on the destination will reject a version it does
  not ship, so on import a versioned type is aliased to its generic form (`q35`
  or `pc`) and the host resolves a supported version. Use `--machine` to pin a
  specific type or `--keep-machine` to preserve the original verbatim.
- **Block-backed disks** are read through `qemu-img` and land as qcow2 files in
  the bundle, which is the intended portable behavior.

## Building the package

From the project root (the directory containing `debian/`):

```
sudo apt build-dep .
dpkg-buildpackage -us -uc -b
```

The resulting `virtpacker_0.3.3_all.deb` is written to the parent directory. The
package is a native Debian package (`3.0 (native)`), so there is no separate
upstream tarball to manage.

## License

MIT. See `debian/copyright`.

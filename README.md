# Xenon (Xbox 360) porting notes
The included [gcc-xenon.patch](gcc-xenon.patch) here adds a mask option to GCC, `-mvmx128`.
When used with `-maltivec`, it disables several instructions missing from Xenon.
I recommend to also use `-mcpu=cell`, because it enables the other instructions.
This should allow a proper 64-bit userland to be built for the Xbox 360!

## Gentoo (with Crossdev)
Place the patch in `/etc/portage/patches/cross-powerpc64-unknown-linux-gnu/gcc/` before running `crossdev --target powerpc64-unknown-linux-gnu`.
I would also recommend copying it to `/usr/powerpc64-unknown-linux-gnu/etc/portage/patches/sys-devel/gcc/` afterwards, so that when GCC itself is cross-compiled later, it retains the patch.

You should then be able to add `-mcpu=cell -maltivec -mvmx128` to CFLAGS.

Unfortunately, Rust is not currently supported for the `powerpc64-unknown-linux-gnu` target. Use a custom profile with WD-40.

If you want a systemd profile, you'll need to build `sys-apps/util-linux` with some temporary USE flags before the system set. `USE="build -systemd -gpm"` should work for both desktop and systemd profiles.

If you want a desktop profile, cross-compilation of gobject-introspection currently proves impossible. `USE="-introspection -vala"` for the system set.

I will assume you want the composition of both for completeness' sake, so your custom profile's `parent` would contain:
```
gentoo:default/linux/ppc64/23.0/systemd
gentoo:default/linux/ppc64/23.0/desktop
gentoo:features/wd40
```

I don't know if just using the existing `desktop/gnome/systemd` profile as a base would be identical to this. It might be, which would partially explain the lack of a separate, dedicated `desktop/systemd` profile.

Some notes about cross-compiling certain packages:

### dev-lang/perl
perl-cross for version `5.42.0` (the latest testing) bails out.

Don't put `~ppc64` in ACCEPT_KEYWORDS for now. You don't need it.

### gnome-base/librsvg
While not a cross-compilation bug, [librsvg-2.40.21_fixup.patch](librsvg-2.40.21_fixup.patch) fixes a build error in version 2.40.21 due to GCC 14 making some warnings into errors by default (see [bug 944392](https://bugs.gentoo.org/944392)), and another build error due to the use of a deprecated header from libxml2.

Copy the patch to `/usr/powerpc64-unknown-linux-gnu/etc/portage/patches/gnome-base/librsvg/`.

### app-crypt/libsecret
Fails cross-compilation if PAM is enabled.

In **/usr/powerpc64-unknown-linux-gnu/etc/portage/package.use**:
```
app-crypt/libsecret -pam
```

### dev-db/sqlite
If ICU support is enabled, sqlite will misconfigure itself and try to use the host's `/usr/include/`. This will cause the build to fail.

In **/usr/powerpc64-unknown-linux-gnu/etc/portage/package.use**:
```
dev-db/sqlite -icu
```

### dev-qt/qtbase
If NLS support is enabled, dev-qt/qttranslations fails to emerge because it can't use Linguist at build time.

In **/usr/powerpc64-unknown-linux-gnu/etc/portage/package.use**:
```
dev-qt/qtbase -nls
```

### kde-frameworks/kguiaddons and kde-frameworks/kwindowsystem
If Wayland support is enabled, kde-frameworks/kguiaddons and kde-frameworks/kwindowsystem won't find wayland.xml properly.

In **/usr/powerpc64-unknown-linux-gnu/etc/portage/package.use**:
```
kde-frameworks/kguiaddons -wayland
kde-frameworks/kwindowsystem -wayland
```

### media-libs/libjpeg-turbo
This package uses compiler intrinsics or something when Altivec is enabled.
It will cause Internal Compiler Errors when used with `-mvmx128`.
This package should be built with `-mno-altivec` as a workaround.

Assuming you kept the default of `-O2 -pipe -fomit-frame-pointer`;

In **/usr/powerpc64-unknown-linux-gnu/etc/portage/env/no-altivec.conf**:
```
CFLAGS="-mcpu=cell -mno-altivec -O2 -pipe -fomit-frame-pointer"
CXXFLAGS="${CFLAGS}"
```

In **/usr/powerpc64-unknown-linux-gnu/etc/portage/package.env**:
```
media-libs/libjpeg-turbo no-altivec.conf
```

### dev-perl/XML-Parser
(This is probably optional)

A check in the ebuild added to fix a different bug fails here.
It tries to run `find "${D}" -name Expat.so | grep Expat || die "Something went badly wrong, can't find Expat.so. Please file a bug."` on the target with QEMU's usermode emulation, which ends up failing due to an endianness mismatch from `libsandbox.so` in LD_PRELOAD.

Copy the ebuild to a local repository, and comment out this check.

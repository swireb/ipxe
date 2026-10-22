## fog-ipxe

- iPXE sources, build configuration and prebuilt binaries for [FOG Project](https://github.com/FOGProject/fogproject).
- FOG boots clients with [iPXE](https://ipxe.org). This repository holds everything FOG layers on top of upstream iPXE — the feature selection, the boot scripts, and the build script — and publishes the resulting binaries as release assets.
- It exists so that fogproject does not have to carry 22 MB of build output in git, and so that "which iPXE is this server running?" has an answer. See [fogproject#959](https://github.com/FOGProject/fogproject/issues/959).

## Most people do not need to build anything

- The FOG installer downloads these binaries from a release, checksums them, and drops them in the TFTP root. That is the whole story for an HTTP install, or an HTTPS install using a publicly trusted certificate.
- There is exactly one reason to build locally: HTTPS with your own CA. The certificate arguments bake that certificate into the binary so iPXE can fetch the boot script over TLS, which makes it a per-server artifact no release can supply. The installer handles this for you when its HTTP protocol setting is HTTPS.

```text
# Certificate arguments
CERT=
TRUST=
# Boot script fetched over TLS
boot.php
# Installer setting
httpproto=https
```

## Layout

- The source, boot scripts, build entry and pinned signed binaries use the following layout.

```text
# BIOS build: config headers and embedded boot scripts
src/
# EFI build: EFI-specific feature selection, config headers only; no embedded script
src-efi/
# The boot script every EFI binary downloads and runs
autoexec.ipxe
# Build entry
buildipxe.sh
# Pinned upstream signed binaries: republished, not built
secureboot/
```

- The BIOS and EFI source trees are near-identical, because most of what differs between them sits inside platform guards and is inert in the other tree.

```text
#if defined ( PLATFORM_* )
```

## The config headers

- These are the FOG-specific part. They are generated, not hand-edited: start from the pristine upstream headers for whatever upstream version is pinned, then apply an explicit table of FOG's deviations, each carrying the reason it exists.
- Do it that way and the next refresh is a re-run rather than a three-way diff, and "is this a FOG choice or is it drift?" is answerable by looking at one table. FOG previously refreshed these by copying upstream's wholesale, which silently disabled the BIOS console for anyone who rebuilt — see [fogproject#958](https://github.com/FOGProject/fogproject/issues/958).

```text
# Pinned upstream version variable
IPXEVER
```

## Building

- Both arguments are optional; defaults are the FOG CA if one is present and the output directory below. Upstream clones land in the build directory.

```shell
./buildipxe.sh [cert] [outdir]
```

```text
# Default output directory
./output
# Upstream clone directory
./build/
```

- The output tree mirrors FOG's TFTP package layout exactly, so it can be copied over a TFTP root unchanged.

```text
# FOG package layout
packages/tftp/
# BIOS with embedded script, x86_64 EFI and autoexec.ipxe
output/
# 32-bit EFI and autoexec.ipxe
output/i386-efi/
# arm64 EFI and autoexec.ipxe
output/arm64-efi/
# BIOS only, with a 10 second pre-DHCP sleep
output/10secdelay/
```

- Requirements: git, make, gcc, binutils, perl, liblzma, mtools, xorriso, and gcc-aarch64-linux-gnu for the arm64 binaries.

## Pinned upstream version

- The upstream clone is pinned to a tag. Bumping it is a deliberate act — FOG spent years tracking master, which meant two people building on the same day could get different binaries.

```shell
IPXEVER=v2.0.1 ./buildipxe.sh
```

## Boot scripts: EFI downloads a script, BIOS embeds one

- No EFI binary here is built with an embedded script. Each one downloads the autoexec script from the directory it was itself loaded from — falling back to the TFTP root — and executes it, because with nothing compiled in the first-image function finds no image ahead of it. Changing the boot logic is therefore editing one text file, with no toolchain and no rebuild. It is also the only shape that works under UEFI Secure Boot, since an embedded script is not permitted in a Secure Boot build. See the [Secure Boot how-to](https://docs.fogproject.org/kb/how-tos/secure-boot-signing/).

```text
# Embedded-script build argument, absent from all EFI targets
EMBED=
# Downloaded EFI script
autoexec.ipxe
# Image selection function
first_image()
```

- A copy of the autoexec script ships in every directory holding an EFI binary. FOG's installer hard-links them so there is exactly one script however many paths reach it.
- Do not put an embedded-script binary in a directory an EMBED-less one can fall back to. An embedded binary still downloads the autoexec script — EFI probing registers it before any driver is connected — but never runs it, so nothing unregisters it and initrd loading concatenates it into the ramdisk ahead of the initramfs. The kernel then panics on the missing compression magic. This is why the embedded-script argument is gone from every EFI target rather than most of them.

```text
# EFI probe and initrd loading functions
efi_probe()
initrd_load_all()
# Initramfs that would be preceded by the downloaded script
init.xz
```

- Legacy BIOS is the exception and keeps an embedded script: it has no EFI autoexec loader, so there is no downloaded script for it to read. That is also why the delayed directory still exists and now holds BIOS files only — on EFI the delay is an installer option that inserts a sleep into the autoexec script.

```text
# Loader absent from legacy BIOS
efi_autoexec_load()
# BIOS delayed directory
10secdelay/
# EFI script delay command
sleep
```

## Secure Boot

- A second release asset carries the pieces a Secure Boot chain needs that FOG cannot build: they have to be signed by keys FOG does not hold — Microsoft's, for the shim, and iPXE's, for the loader it chains to.

```text
# Secure Boot release asset
fog-ipxe-secureboot-<tag>.tar.gz
# ipxe/shim, signed by Microsoft (2011 and 2023)
secureboot/snponly-shimx64.efi
# Upstream's signed iPXE
secureboot/snponly.efi
# MokManager, for enrolling your own key
secureboot/mmx64.efi
# This repository's own boot script
secureboot/autoexec.ipxe
# The same set for arm64
secureboot/arm64-efi/…
# Origin and hashes
secureboot/MANIFEST
```

- Everything but the autoexec script is upstream's, republished byte for byte. The upstream lock pins the release tags and the sha256 of every file; the staging script fetches them, checks those hashes, and reads the PE certificate table back to assert each binary is signed by the key it should be. A mismatch fails the release rather than shipping — an unsigned or test-signed binary that reaches an install surfaces at the client as "Security Policy Violation" with nothing server-side to explain it.

```text
# Upstream lock and staging script
secureboot/upstream.lock
secureboot/stage.sh
```

- Staging runs on every build, not just tags, so a pull request proves the pins still resolve.
- The shim filename is a rename, and the name is the mechanism. Shim picks its second stage by rewriting its own filename suffix, so this one loads the SNP-only loader. Under shim's stock name it would load the all-drivers build, which hangs wherever native NIC takeover fails. The suffix match allows the shim marker plus at most four more characters, and the arm64 suffix is exactly at that limit: do not lengthen these names.

```text
# Renamed shim and the second-stage loader it selects
snponly-shimx64.efi
snponly.efi
# Suffix rewrite
-shim<arch>.efi -> .efi
# The all-drivers build selected under shim's stock name
ipxe.efi
# Shim marker and arm64 suffix at the four-character limit
-shim
-shimaa64
```

- The autoexec script must sit beside the binaries, because iPXE resolves the bare name against its current working URI.

```text
# Script and current working URI identifier
autoexec.ipxe
cwuri
```

- A Secure Boot chain still needs one thing this release cannot supply: the FOS kernel is unsigned, so it has to be signed with your own key and that key enrolled via MokManager. See the [Secure Boot how-to](https://docs.fogproject.org/kb/how-tos/secure-boot-signing/).
- Bumping a pin is a reviewable one-file diff. The release variable is asserted against the build version in the build script rather than derived from it — if the two drift, Secure Boot clients run a different iPXE release from every other client on the same server.

```text
# Asserted release and build version variables
IPXE_RELEASE
IPXEVER
# Assertion location
buildipxe.sh
```

## Offline installs

- Pre-place this checkout — and its build clones — at the installer source path below. The build script reuses an existing clone rather than fetching, so a machine with no internet access can still build.

```text
# Installer source path and its default
$fogprogramdir/ipxe
/opt/fog/ipxe
# Clone directory and build entry
build/
buildipxe.sh
```

## Licence

- iPXE is GPLv2 (with a UBDL option); see upstream. The FOG-specific files here are GPLv3, matching fogproject.

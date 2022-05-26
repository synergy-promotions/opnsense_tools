About the OPNsense tools
========================
*(This fork's instructions are for building to target the Raspberry Pi 3B and 3B + boards)*

In conjunction with src.git, ports.git, core.git and plugins.git they
create sets, packages and images for the OPNsense project.

http://ftp.freebsd.org/pub/FreeBSD/ports/ports/README.TXT

http://ftp.freebsd.org/pub/FreeBSD/releases/

https://opnsense.c0urier.net/releases/22.1/

Setting up a build system
=========================

(Makefile -- branch and version set to 22.1)

** Must use PKG version 1.16.3 not the latest branch **

Install on a FreeBSD 13.1 (amd64) virtual machine with at least 30GB of hard disk (UFS works better than ZFS) and at least 4GB of RAM to successfully build all standard images.
All tasks require a root user.  Do the following to grab the repositories
(overwriting standard ports and src):

    # pkg update && pkg install -y git nano curl subversion aarch64-binutils qemu-user-static u-boot-rpi3 rpi-firmware
    # cd /usr/share
    # git clone https://github.com/synergy-promotions/opnsense_tools tools
    # cd tools
    # make update OS=13.1
    
Now all the core components should be configured. You can build the device and target architecture images like so:
    
<b> e.g. to build images for RPI3...
    
    # make xtools base kernel packages arm-3G DEVICE=RPI3 OS=13.1
    # make -d a -j4 xtools base kernel packages arm-3g DEVICE=RPI3 SETTINGS=22.1 OS=13.1 PRODUCT_ARCH=aarch64 PRODUCT_TARGET=aarch64
</b>

DVD ISO
=====

    # make dvd

If successful, a dvd image can be found under:

    # make print-IMAGESDIR

Detailed build steps and options
================================

How to specify build options on the command line
------------------------------------------------

The build is broken down into individual stages: base,
kernel, ports, plugins and core can be built separately and
repeatedly without affecting the other stages.  All stages
can be reinvoked and continue building without cleaning the
previous progress.  A final stage assembles all five stages
into a target image.

All build steps are invoked via make(1):

    # make step OPTION="value"

Available early build options are:

* SETTINGS:	the name of the requested local configuration
* CONFIGDIR:	read configuration from other directory and override SETTINGS

Available build options are:

* ABI:		a custom ABI (defaults to SETTINGS)
* ADDITIONS:	a list of packages/plugins to add to images
* ARCH:		the target architecture if not native
* COMPORT:	serial port, e.g. "0x3f8" (default)
* COMSPEED:	serial speed, e.g. "115200" (default)
* DEVICE:	loads device-specific modifications, e.g. "A10" (default)
* FLAVOUR:	"OpenSSL" (default), "LibreSSL", "Base"
* KERNEL:	the kernel config to use, e.g. SMP (default)
* MIRRORS:	a list of mirrors to prefetch sets from
* NAME:		"OPNsense" (default)
* PRIVKEY:	the private key for signing sets
* PUBKEY:	the public key for signing sets
* SUFFIX:	the suffix of top package name (default is empty)
* TYPE:		the base name of the top package to be installed
* UEFI:		use amd64 hybrid images for said images, e.g. "vga vm"
* VERSION:	a version tag (if applicable)

How to specify build options via configuration file
---------------------------------------------------

The configuration file is required at "CONFIGDIR/SETTINGS/build.conf".
Its contents can be modified to adapt a non-standard build environment
and to avoid excessive Makefile arguments.  Use with care.

How to run individual or composite build steps
----------------------------------------------

Kernel, base, packages and release sets are stored under:

    # make print-SETSDIR

All final images are stored under:

    # make print-IMAGESDIR

Build the userland binaries, bootloader and administrative files:

    # make base

Build the kernel and loadable kernel modules:

    # make kernel

Build all the third-party ports:

    # make ports

Build additional plugins if needed:

    # make plugins

Wrap up our core as a package:

    # make core

A dvd live image is created using:

    # make dvd

A serial memstick live image is created using:

    # make serial

A vga memstick live image is created using:

    # make vga

A flash card full disk image is created using:

    # make nano

A virtual machine full disk image is created using:

    # make vm

Release sets can be built as follows although the result is
an unpredictable set of images depending on the previous
build states:

    # make release

However, the release target is necessary for the following
target which includes sanity checks, proper clearing of the
images directory and core package version alignment:

    # make distribution

Cross-building for other architecures
-------------------------------------
All in one go:

    # make xtools base kernel packages arm-3G DEVICE=RPI3
 
This feature is currently experimental and requires installation
of packages for cross building / user mode emulation and additional
boot files to be installed as prompted by the build system.

A cross-build on the operating system sources is executed by
specifying the target architecture and custom kernel:

    # make base kernel DEVICE=RPI3

In order to speed up building of using an emulated packages build,
the xtools set can be created like so:

    # make xtools DEVICE=RPI3

The xtools set is then used during the packages build similar to
the distfiles set.

    # make packages DEVICE=RPI3

The final image is built using:
    
    # make arm-<size> DEVICE=RPI3

Currently available device are: BANANAPI and RPI2, RPI3.

About other scripts and tweaks
==============================

Device-specific settings
------------------------

Device-specific settings can be found and added in the
device/ directory.  Of special interest are hooks into
the build process for required non-default settings for
image builds.  The .conf files are shell scrips that can
define hooks in the form of e.g.:

    serial_hook()
    {
        # ${1} is the target file system root
        touch ${1}/my_custom_file
    }

These hooks are available for all image types, namely
dvd, nano, serial, vga and vm.  Device-specific hooks
are loaded after config-specific hooks and both of them
can coexist in a given build.

Updating the code repositories
------------------------------

Updating all or individual repositories can be done as follows:

    # make update[-<repo1>[,...]]

Available update options are: core, plugins, ports, portsref, src, tools

Regression tests and ports audit
--------------------------------

Before building images, you can run the regression tests
to check the integrity of your core.git modifications plus
generate output for the style checker:

    # make test

To check the binary packages from ports against the upstream
vulnerability database run the following:

    # make audit

Advanced package builds
-----------------------

Package sets ready for web server deployment are automatically
generated and modified by ports, plugins and core setps.  The
build automatically caches temporary build dependencies to avoid
spurious rebuilds.  These packages are later discarded to provide
a slim runtime set only.

If signing keys are available, the packages set will be signed
twice, first embedded into repository metadata (inside) and
then again as a flat file (outside) to ensure integrity.

For faster ports building it may be of use to cache all distribution
files before running the actual build:

    # make distfiles

For targeted rebuilding of already built packages the following
works:

    # make ports-<packagename>[,...]
    # make plugins-<packagename>[,...]
    # make core-<packagename>[,...]

Please note that reissuing ports builds will clear plugins and
core progress.  However, following option apply to PORTSENV:

* DEPEND=no	Do not tamper with plugins or core packages
* PRUNE=no	Do not check ports integrity prior to rebuild
* SILENT=no	Do not use make(1) -s command line option

The defaults for these ports options are set to "yes".  A sample
invoke is as follows:

    # make ports-openssl PORTSENV="DEPEND=no PRUNE=no"

Acquiring precompiled sets from the mirrors or another local direcory
---------------------------------------------------------------------

Compiled sets can be prefetched from a mirror if they exist,
while removing any previously available set:

    # make prefetch-<option>[,...] [VERSION=<full_version>]

If another build configuration is used locally that is compatible,
the sets can be cloned from there as well:

    # make clone-<option>[,...] TO=<major_version>

Available prefetch or clone options are:

* base:		select matching base set
* distfiles:	select matching distfiles set (clone only)
* kernel:	select matching kernel set
* packages:	select matching packages set

Using signatures to verify integrity
------------------------------------

Signing for all sets can be redone or applied to a previous run
that did not sign by invoking:

    # make sign-base,kernel,packages

A verification of all available set signatures is done via:

    # make verify

Nano image size adjustment
--------------------------

Nano images can be adjusted in size using an argument as follows:

    # make nano-<size>

Virtual machine images
----------------------

Virtual machine images come in varying disk formats and sizes.
For this reason they are not included in our binary releases.
The default format is vmdk with 20G and 1G swap.  If you want
to change that you can manually alter the invoke using:

    # make vm-<format>[,<size>[,<swap>]]

Available virtual machine disk formats are:

* qcow:		Qemu, KVM (legacy format)
* qcow2:	Qemu, KVM (not backwards-compatible)
* raw:		Unformatted (sector by sector)
* vhd:		VirtualPC, Hyper-V, Xen (dynamic size)
* vhdf:		Azure, VirtualPC, Hyper-V, Xen (fixed size)
* vmdk:		VMWare, VirtualBox (dynamic size)

The swap argument is either its size or set to "off" to disable.

Clearing individual build step progress
---------------------------------------

A couple of build machine cleanup helpers are available
via the clean script:

    # make clean-<option>[,...]

Available clean options are:

* arm:		remove arm image
* base:		remove base set
* distfiles:	remove distfiles set
* dvd:		remove dvd image
* core:		remove core from packages set
* images:	remove all images
* kernel:	remove kernel set
* logs:		remove all logs
* nano:		remove nano image
* obj:		remove all object directories
* packages:	remove packages set
* plugins:	remove plugins from packages set
* ports:	alias for "packages" option
* release:	remove release set
* serial:	remove serial image
* sets:		remove all sets
* src:		reset kernel/base build directory
* stage:	reset main staging area
* vga:		remove vga image
* vm:		remove vm image
* xtools:	remove xtools set

How the port tree is synced with its upstream repository
--------------------------------------------------------

The ports tree has a few of our modifications and is sometimes a
bit ahead of HardenedBSD.  In order to keep the local changes, a
skimming script is used to review and copy upstream changes:

    # make skim[-<option>]

Available options are:

* used:		review and copy upstream changes
* unused:	copy unused upstream changes
* (none):	all of the above

Rebasing the file lists for the base sets
-----------------------------------------

In case base files changed, the base package list and obsoleted
files need to be regenerated.  This is done using:

    # make rebase

Switching to the build jail for inspection
------------------------------------------

Shall any debugging be needed inside the build jail, the following
command will use chroot(8) to enter the active build jail:

    # make chroot[-<subdir>]

Boot images in the native bhyve(8) hypervisor
---------------------------------------------

There's also the posh way to boot a final image using bhyve(8):

    # make boot-<image>

Please note that the system does not have working networking after
bootup and login is only possible via the Nano and Serial images.

Generating a make.conf for use in running OPNsense
--------------------------------------------------

A ports tree in a running OPNsense can be used to build packages
not published on the mirrors.  To generate the make.conf contents
for standalone use on the host use:

    # make make.conf

Reading and modifying version numbers of build sets and images
--------------------------------------------------------------

Normally the build scripts will pick up version numbers based
on commit tags or given version tags or a date-type string.
Should it not fit your needs, you can change the name using:

    # make rename-<set>[,<another_set>] VERSION=<new_name>

The available targets are: base, distfiles, dvd, kernel, nano,
packages, serial, vga and vm.

The current state of the associated build repositories checked
out on the system can be printed using:

    # make info

Repositories that have signing keys can show the current
fingerprint using:

    # make fingerprint

Last but not least, in case build variables needs to be inspected,
they can be printed selectively using:

    # make print-<variable1>[,<variable2>]

Compressing images
------------------

Images are compressed using bzip2(1) for distribution.  This can
be invoked manually using:

    # make compress-<image1>[,<image2>]

Composite build steps
---------------------

Build steps are pinned to a particular crypto flavour, but if OpenSSL
and LibreSSL packages are both required they can be batch-built using:

    # make batch-<step>[,<option>[,...]]

A fully contained nightly build for the system is invoked using:

    # make nightly

To allow the nightly build to build both release and development packages
use:

    # make nightly EXTRABRANCH=master

Nightly builds are the only builds that write and archive logs under:

    # make print-LOGSDIR

with ./latest containing the last nightly build run.  Older logs are
archived and available for a whole week for retrospective analysis.

To push sets and images to a remote location use the upload target:

    # make upload-<set>[,...]

To pull sets and images from a remote location use the download target:

    # make download-<set>[,...]

Logs can be downloaded as well for local inspection.  Note that download
like prefetch will purge all locally existing targets.  Use SERVER to
specify the remote end, e.g. SERVER=user@does.not.exist

Additionally, UPLOADDIR can be used to specify a remote location.  At
this point only "logs" upload cleares and creates directories on the fly.

If you want to script interactive prompts you may use the confirm target
to operate yes or no questions before an action:

    # make info confirm dvd

Last but not least, a rebuild of OPNsense core and plugins on package
sets is invoked using:

    # make hotfix[-<step>]

It will flush all previous packages except for ports, rebuild core and
plugins and sign the sets if enabled.  It can also explicity set "core"
or "plugins".

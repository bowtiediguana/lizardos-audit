# lizardos-audit
Instructions for auditing LizardOS, links to third party audits

LizardOS is built on Qubes OS, which is endorsed by NSA whistleblower Edward Snowden and other cybersecurity experts and privacy advocates.

For the purpose of this audit we assume you trust the Qubes developers and would be comfortable to run vanilla Qubes. If you do not trust an official Qubes release, then you can't trust LizardOS either.

To audit LizardOS you will need at least 40GB of free disk space and a trusted Fedora Linux install. We suggest renting a fast cloud server running Fedora to rapidly perform this audit from a trusted temporary environment.

Here are the steps:

1. Purchase LizardOS and download the ISO
2. Obtain the Qubes release LizardOS was built on
3. Verify the signatures / hashes of both ISOs
4. Extract both ISOs
5. Perform a diff to focus only on the files which are different in LizardOS
6. Inspect the differences - you'll need to read Python, Bash, and Salt files, and extract some RPMs

## Extract LizardOS

The first beta release of LizardOS is called `LizardOS-Beta-1.iso` and has sha256 hash `58cad5c0670066912157164441c294bd05542ed3372a02b254e6d2e7c9e6edc4`.

On a trusted Fedora Linux computer, download this ISO and place it in a new directory called `audit`.

If you are using a cloud server, you may need to `scp` the file from your local machine where you downloaded it, to the audit directory on the Fedora cloud server.

Verify the hash:

```bash
[ "$(sha256sum LizardOS-Beta-1.iso | awk '{print $1}')" == "58cad5c0670066912157164441c294bd05542ed3372a02b254e6d2e7c9e6edc4" ] && echo "Hashes match" || echo "Hashes do not match"
```

Extract the ISO
```bash
command -v xorriso >/dev/null 2>&1 || sudo dnf install xorriso -y # install a package to extract ISO's from the Fedora repository
mkdir lizardos
xorriso -osirrox on -indev LizardOS-Beta-1.iso  -- -extract / lizardos/
```

## LizardOS Structure

The `lizard` directory is not present in QubesOS.


```
├── anaconda
│   └── ui
│       └── spokes
├── postinstall
│   ├── 3pty
│   │   ├── Qubes-vpn-support
│   │   │   ├── files-main
│   │   │   │   ├── qubes-vpn-handler.service.d
│   │   │   │   └── vpn
│   │   │   └── tests
│   │   │       ├── openvpn-passwordless
│   │   │       │   ├── client
│   │   │       │   │   └── rw-config-vpn
│   │   │       │   └── server
│   │   │       │       └── sample-keys
│   │   │       │           └── sample-ca
│   │   │       └── wg-mullvad
│   │   ├── ledgerlive
│   │   └── qvm-create-windows-qube
│   │       ├── icons
│   │       ├── post
│   │       ├── tools-media
│   │       │   └── auto-qwt
│   │       │       └── driver-certificates
│   │       └── windows-media
│   │           ├── answer-files
│   │           ├── isos
│   │           └── out
│   ├── keys
│   ├── mime
│   ├── salt
│   ├── skel
│   │   └── Desktop
│   ├── theme
│   └── win
└── pyanaconda
    ├── core
    ├── modules
    │   ├── common
    │   │   └── structures
    │   └── users
    └── ui
        └── gui
            └── spokes

```


### Understanding the code

**anaconda** and **pyanaconda**
Mostly Python, controls the Anaconda Installer, pay particular attention to the `cryptsetup` commands which control the full disk encryption. Can verify key isn't exfiltrated.

**postinstall**
Root directory contains RPM files: the Linux kernel, and Qubes templates for Debian 11, Debian 11 Minimal, and Whonix 16. These are unmodified and hashes can be compared with official sources.

`3pty` contains binaries (rpm and deb) from package issuers including Brave and Opera browsers, Protonmail bridge, and others which are not part of the official Fedora or Debian repositories. These are unmodified and hashes can be compared with official sources. `Qubes-vpn-support` and `qvm-create-windows-qube` are made by the Qubes community. These have been modified for the current project. These are small tools written in Bash and are easy to verify as not malicious. They automate the process of configuring a VPN and installing Windows respectively.

`win/get-win-server.sh` downloads a windows ISO from a `.microsoft.com` and removes the download immediately if it doesn't match a known hash. Hashes can be verified by downloading an ISO from Microsoft on a trusted computer and computing the hash.

`keys` contains third party keys, currently only for Brave, to ensure that packages downloaded through those new repositories are signed by the publisher.

`mime` contains textfiles for file handlers and mime defaults. `open-in-dvm.desktop` is a special handler which passes the file to a new sandboxed temporary VM.

`salt` contains a number of configuration files in SaltStack format. Their only purpose is to ensure required packages are installed in template VMs.

`theme` contains a textfile to configure GTK to use dark mode by default

`skel` contains:
 - `Desktop` - desktop shortcuts / launchers for the post install and log collection tools
 - `.config` - various Linux "dotfiles" (TODO: some of which may not be needed), mainly for configuring the Anon / Personal / Secure desktops on KDE and the KWin rules for automatically opening VM apps in the appropriate activity/context. These are ASCII text, with 1 sqlite database. (kactivitymanagerd) There is a binary file, `database-shm`. (TODO: check if this can be removed)

 **postinstall Bash scripts**
 `brave.sh`, `libreoffice.sh` perform basic configuration of dotfiles for these apps.
 `windows.sh` installs Windows, `whonix.sh` installs whonix.
 `ledger.sh` installes Ledger Live from official sources.
 `sendlogs.sh` uploads logfiles from this postinstall folder to a pastebin, and outputs the pastebin URL. The intended purpose is to share install log data for troubleshooting purposes. The user can access the URL prior to sharing to see what is shared.

 The main work is done with `postinstall.sh`, this is called by the user by desktop shortcut after they connect to the Internet.

 These files need to be carefully inspected for security issues, for example verifying the location and hashes of the Ledger Live download.

## Download and extract QubesOS

LizardOS is built on Qubes 4.1.

```bash
command -v aria2c >/dev/null 2>&1 || sudo dnf install aria2 -y # install aria download software for faster download
aria2c -x 12 -s 4 https://mirrors.edge.kernel.org/qubes/iso/Qubes-R4.1.2-x86_64.iso
```

Check the hash (cross reference the digests here: https://mirrors.edge.kernel.org/qubes/iso/Qubes-R4.1.2-x86_64.iso.DIGESTS)

```bash 
[ "$(sha256sum Qubes-R4.1.2-x86_64.iso | awk '{print $1}')" == "4240897c83882329c960863da18125badc9e4a4a1071340240577c70aef5dcd8" ] && echo "Hashes match" || echo "Hashes do not match"
```

Extract the ISO

```bash
mkdir qubesos
xorriso -osirrox on -indev Qubes-R4.1.2-x86_64.iso  -- -extract / qubesos/
```

## Perform a diff

```bash
diff -qr qubesos/ lizardos/ | tee diff.txt && sort diff.txt #perform a recursive diff 
```

### Different boot images

```
Files qubesos/.discinfo and lizardos/.discinfo differ
Files qubesos/.treeinfo and lizardos/.treeinfo differ
Files qubesos/EFI/BOOT/grub.cfg and lizardos/EFI/BOOT/grub.cfg differ
Files qubesos/LiveOS/squashfs.img and lizardos/LiveOS/squashfs.img differ
Files qubesos/images/efiboot.img and lizardos/images/efiboot.img differ
Files qubesos/images/pxeboot/initrd-latest.img and lizardos/images/pxeboot/initrd-latest.img differ
Files qubesos/images/pxeboot/initrd.img and lizardos/images/pxeboot/initrd.img differ
Files qubesos/images/pxeboot/vmlinuz and lizardos/images/pxeboot/vmlinuz differ
Files qubesos/images/pxeboot/vmlinuz-latest and lizardos/images/pxeboot/vmlinuz-latest differ
Files qubesos/images/pxeboot/xen.gz and lizardos/images/pxeboot/xen.gz differ
Files qubesos/isolinux/boot.cat and lizardos/isolinux/boot.cat differ
Files qubesos/isolinux/grub.conf and lizardos/isolinux/grub.conf differ
Files qubesos/isolinux/initrd.img and lizardos/isolinux/initrd.img differ
Files qubesos/isolinux/isolinux.bin and lizardos/isolinux/isolinux.bin differ
Files qubesos/isolinux/isolinux.cfg and lizardos/isolinux/isolinux.cfg differ
Files qubesos/isolinux/splash.png and lizardos/isolinux/splash.png differ
Files qubesos/isolinux/vmlinuz and lizardos/isolinux/vmlinuz differ
Files qubesos/isolinux/xen.gz and lizardos/isolinux/xen.gz differ
Files qubesos/repodata/repomd.xml and lizardos/repodata/repomd.xml differ
```

The text and image files can be trivially inspected. For the kernels, it's a matter of obtaining another copy of the same version and comparing hashes.

### Files Only in LizardOS

```
Only in lizardos/: ks.cfg
Only in lizardos/: lizard
Only in lizardos/EFI/BOOT: BOOTX64.cfg
```

The lizard/ directory is explained in the "Understanding the code" section above.
`ks.cfg` is an anaconda kickstarter file to automate installation. It's a plain text human readable file.

`BOOTX64.cfg` is a human readable text file used by the boot loader.


```
Only in lizardos/isolinx: initrd/vmlinuz
```

These are Linux kernel images.


#### Verifying RPM Packages

```
Only in lizardos/Packages: *rpm
```

LizardOS is built with the latest packages, these will differ from the versions included in the Qubes ISO point release. However, they can be obtained from the official Qubes/Fedora repositories over HTTPS.

This snippet installs the Fedora 32 and Qubes repositories, downloads the package versions found in the LizardOS ISO, and compares the trusted downloads with the versions appearing in the ISO.

```bash

#move all the RPMs, includeing from the lizard/ folder, into the Packages folder for comparison
find lizardos/ -type f -name "*.rpm" -exec mv {} lizardos/Packages/ \;

find lizardos/ -type f -name "*.rpm" -exec basename {} \; | sort > get-rpms.txt 
cat get-rpms.txt | sed  's/.rpm$//g' > rpm_names.txt

cat << EOF > fedora32.repo
[fedora32]
name=Fedora 32 - x86_64
#baseurl=http://download.example/pub/fedora/linux/releases/32/Everything/x86_64/os/
metalink=https://mirrors.fedoraproject.org/metalink?repo=fedora-32&arch=x86_64
enabled=1
countme=1
metadata_expire=7d
repo_gpgcheck=0
type=rpm
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-fedora-32-x86_64
skip_if_unavailable=False
EOF

cat << EOF > fedora-updates32.repo
[fedora-updates32]
name=Fedora 32 - x86_64 - Updates
#baseurl=http://download.example/pub/fedora/linux/updates/32/Everything/x86_64/
metalink=https://mirrors.fedoraproject.org/metalink?repo=updates-released-f32&arch=x86_64
enabled=1
countme=1
repo_gpgcheck=0
type=rpm
gpgcheck=1
metadata_expire=6h
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-fedora-32-x86_64
skip_if_unavailable=False
EOF

cat << EOF > qubes-dom0.repo
[qubes-dom0-current]
name = Qubes Dom0 Repository (updates)
#baseurl = https://yum.qubes-os.org/r4.1/current/dom0/fc32
metalink = https://yum.qubes-os.org/r4.1/current/dom0/fc32/repodata/repomd.xml.metalink
skip_if_unavailable=False
enabled = 1
metadata_expire = 6h
gpgcheck = 1
gpgkey = file:///etc/pki/rpm-gpg/RPM-GPG-KEY-qubes-32-primary

[qubes-dom0-current-testing]
name = Qubes Dom0 Repository (updates-testing)
#baseurl = https://yum.qubes-os.org/r4.1/current-testing/dom0/fc32
#baseurl = http://yum.qubesosfasa4zl44o4tws22di6kepyzfeqv3tg4e3ztknltfxqrymdad.onion/r4.1/current-testing/dom0/fc32
metalink = https://yum.qubes-os.org/r4.1/current-testing/dom0/fc32/repodata/repomd.xml.metalink
skip_if_unavailable=False
enabled = 0
metadata_expire = 6h
gpgcheck = 1
gpgkey = file:///etc/pki/rpm-gpg/RPM-GPG-KEY-qubes-r4.1-primary

[qubes-templates-itl]
name = Qubes Templates repository
#baseurl = https://yum.qubes-os.org/r4.1/templates-itl
#baseurl = http://yum.qubesosfasa4zl44o4tws22di6kepyzfeqv3tg4e3ztknltfxqrymdad.onion/r4.1/templates-itl
metalink = https://yum.qubes-os.org/r4.1/templates-itl/repodata/repomd.xml.metalink
enabled = 1
fastestmirror = 1
metadata_expire = 7d
gpgcheck = 1
gpgkey = file:///etc/qubes/repo-templates/keys/RPM-GPG-KEY-qubes-r4.1-primary

[qubes-templates-itl-testing]
name = Qubes Templates repository
#baseurl = https://yum.qubes-os.org/r4.1/templates-itl-testing
#baseurl = http://yum.qubesosfasa4zl44o4tws22di6kepyzfeqv3tg4e3ztknltfxqrymdad.onion/r4.1/templates-itl-testing
metalink = https://yum.qubes-os.org/r4.1/templates-itl-testing/repodata/repomd.xml.metalink
enabled = 0
fastestmirror = 1
gpgcheck = 1
gpgkey = file:///etc/qubes/repo-templates/keys/RPM-GPG-KEY-qubes-r4.1-primary

[qubes-templates-community]
name = Qubes Community Templates repository
#baseurl = https://yum.qubes-os.org/r4.1/templates-community
#baseurl = http://yum.qubesosfasa4zl44o4tws22di6kepyzfeqv3tg4e3ztknltfxqrymdad.onion/r4.1/templates-community
metalink = https://yum.qubes-os.org/r4.1/templates-community/repodata/repomd.xml.metalink
enabled = 0
fastestmirror = 1
metadata_expire = 7d
gpgcheck = 1
gpgkey = file:///etc/qubes/repo-templates/keys/RPM-GPG-KEY-qubes-r4.1-templates-community

[qubes-templates-community-testing]
name = Qubes Community Templates repository
#baseurl = https://yum.qubes-os.org/r4.1/templates-community-testing
#baseurl = http://yum.qubesosfasa4zl44o4tws22di6kepyzfeqv3tg4e3ztknltfxqrymdad.onion/r4.1/templates-community-testing
metalink = https://yum.qubes-os.org/r4.1/templates-community-testing/repodata/repomd.xml.metalink
enabled = 0
fastestmirror = 1
gpgcheck = 1
gpgkey = file:///etc/qubes/repo-templates/keys/RPM-GPG-KEY-qubes-r4.1-templates-community
EOF

sudo mv fedora32.repo fedora-updates32.repo qubes-dom0.repo /etc/yum.repos.d/
sudo dnf install dnf-plugins-core

#Remove some modified RPMs from the comparison, we will deal with these separately
#These are graphics files which were updated with LizardOS Branding
sed -i '/grub2-qubes-theme-5.14.4-5.fc32.x86_64/d' rpm_names.txt
sed -i '/qubes-artwork-4.2.1-1.fc32.noarch/d' rpm_names.txt
sed -i '/qubes-artwork-anaconda-4.2.1-1.fc32.noarch/d' rpm_names.txt
sed -i '/qubes-artwork-plymouth-4.2.1-1.fc32.noarch/d' rpm_names.txt

#This kernels aren't in the repos, pull them manually
grep kernel-latest rpm_names.txt
sed -i '/kernel-latest/d' rpm_names.txt
wget -nc https://ftp.qubes-os.org/repo/yum/r4.1/current-testing/dom0/fc32/rpm/kernel-latest-6.0.7-1.fc32.qubes.x86_64.rpm -O trusted-rpms/kernel-latest-6.0.7-1.fc32.qubes.x86_64.rpm
wget -nc https://ftp.qubes-os.org/repo/yum/r4.1/current-testing/dom0/fc32/rpm/kernel-latest-qubes-vm-6.0.7-1.fc32.qubes.x86_64.rpm -O trusted-rpms/kernel-latest-qubes-vm-6.0.7-1.fc32.qubes.x86_64.rpm

mkdir -p trusted-rpms
dnf --disablerepo="*" --enablerepo="fedora*32" --enablerepo="qubes*" download --destdir=trusted-rpms $(cat rpm_names.txt)


#Clean up the repos we installed
pushd /etc/yum.repos.d/
sudo rm fedora32.repo fedora-updates32.repo qubes-dom0.repo
popd
```

Say yes if prompted to import a Qubes signing key.

```bash
diff -rq lizardos/Packages/ trusted-rpms/ | sort | tee differences.txt
```

Verify no differences except for this output:

```
Only in lizardos/Packages/: grub2-qubes-theme-5.14.4-5.fc32.x86_64.rpm
```

Finally, inspect the above RPM. We will extract the contents to verify they are not malicious.

```bash
mkdir -p edited/grub2-qubes-theme-5.14.4-5.fc32.x86_64.rpm
rpm2cpio lizardos/Packages/grub2-qubes-theme-5.14.4-5.fc32.x86_64.rpm | (cd edited/grub2-qubes-theme-5.14.4-5.fc32.x86_64.rpm && cpio -idmv)
```
Inspect all files under the `edited` directory and compare with the version of `grub2-qubes-theme` in `qubesos/Packages`. The change is an `*svg` file with the LizardOS branding.

```
Only in lizardos/repodata: *
```

These are Redhat Package Manager / DNF package lists. They can be uncompressed and inspected as XML files.

### Files Only in QubesOS

The differences are earlier package versions, kernels, and different repodata. These don't need to be inspected as 1) they come from Qubes and 2) they are not present in LizardOS.

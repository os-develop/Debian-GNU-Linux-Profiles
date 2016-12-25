## Ways to build your own trustchain for secureboot.
##### Copyright (c) TYA
##### Homepage: http://tya.company/

##### Background

Secure Boot need a trustchain, which has been described in [Pollux's blog](https://www.wzdftpd.net/blog/uefi-secureboot-debian.html) and [James Bottomley's random Pages](http://blog.hansenpartnership.com/owning-your-windows-8-uefi-platform/). This scheme is even useful for virtual machine, if [OVMF](http://www.tianocore.org/ovmf/) is used as the boot firmware.

##### File based trustchain building.

The following packages should be installed:

```
# apt-get install gnutls-bin uuid-runtime efitools sbsigntool
```

In the blogs above there have been some commands to do the deed, but I believe, in order to resolve these structural dependency, `make(1)` should be used to ease the execution, so I wrote [this Makefile](../scripts/secureboot/Makefile) to build the trustchain. Although different keys had better be held in different administrators, they can all use this Makefile to perform their own duty.

To create certificates in batch, template files are needed, some exemplative files are provided in [this directory](../scripts/secureboot/)

Run `make auth` to create .auth files which UEFI with secure boot feature accepts, and `make signedtools` could be used to create signed version of efi executables privided by `efitools` package, which are needed to import and manipulate keys on UEFI shell.

##### Test on virtual machine.

Create a disk image with the following GPT table via `gdisk(8)` (no need to be accurate):

```
Number  Start (sector)    End (sector)  Size       Code  Name
   1              34            2048   1007.5 KiB  EF02  BIOS boot partition
   2            2074          131038   63.0 MiB    EF00  EFI System
```

Bind the image to a loop device:

`$ udisksctl loop-setup -f efiboot.img`

Create a fat32 file system on the second partition:

`# mkfs.vfat -n EFIBOOT /dev/loop0p2`

and mount it:

`$ udisksctl mount -b /dev/loop0p2`

Symlink its mount point to your working directory, or change the value of `DISKPATH` variable within the Makefile.

Run `make install`. .auth files will be installed to the root of the file system with the image, while efi executables (signed and unsigned) will be installed to EFT/BOOT/.

Then you can test this image on a libvirt virtual machine using OVMF as boot firmware. Using `KeyTool(.signed).efi` (whose usage is described inJames' blog above) to manipulate secure boot variables. If failed, reboot the VM, and press ESC to enter the config interface to remove the keys.

The nopk.auth created from /dev/null could be used to delete pk signed with the same cert.

The image could also be used on physical machine if written to a usb drive.

##### Experimental PKCS#11 based trustchain building (not fully working).

Private keys should be stored in hardware module (e.g. PKCS#11 ones), so I developed the [modified Makefile](../scripts/secureboot/Makefile.p11). 

Unfortunately, this scheme has not yet been able to produce valid signed efi executable, as well as usable nopk.auth, only the main trustchain is valid. I hope someone could help us to improve this scheme.

Assuming you guys have already had enough 2048-bit RSA keys stored in a smart card-like device supported by [OpenSC](https://github.com/OpenSC/OpenSC), you have to config your GnuTLS for PKCS#11 first:

	$ mkdir -p $HOME/.config/pkcs11/modules/
	$ echo 'module: /usr/lib/x86_64-linux-gnu/opensc-pkcs11.so' > $HOME/.config/pkcs11/modules/opensc.module
	
Then list the PKCS#11 urls of your keys with the following command:

	$ p11tool --provider opensc-pkcs11.so --list-privkeys --login --only-urls

write them down (one line per key)to the .url files needed by Makefile, one key per file, as targets of `make(1)`.

In order to run `certtool(1)` in batch mode, the `PIN`(`GNUTLS_PIN`) environment variable should be set to the real pin of your card:

	$ PIN=<pin of card> make (...)

Some certificates should be imported back to the card. Warning: the label of a key may change after the corresponding certificate is imported, so do its PKCS#11 url. If so, you may have to write down its new url to corresponding .url file again.

Rather than invoking `sign-efi-sig-list(1)` directly, in order to use PKCS#11 module, `sign-efi-sig-list(1)` is used to export payloads to sign as files, then sign them with `certtool(1)`, finally, `sign-efi-sig-list(1)` is invoked again to assemble signatures with efi signature lists to produce .auth files.

Note the command to generate `timestamp`, sign-efi-sig-list use `strptime(3)` with "`%c`" to parse the timestamp string. It is very apt to fail if other time format is used, producing invalid .auth files.

There seems only libNSS-based `pesign(1)` able to sign efi executables with PKCS#11 modules, so a certdb directory should be create first.

######Reference: 
######[1] man page of sign-efi-sig-list, sbsign and pesign
######[2] man page of certtool and smime (1ssl)
######[3] Source code of sign-efi-sig-list
######[4] [User:Pjones/SecureBootSmartCardDeployment](https://fedoraproject.org/wiki/User:Pjones/SecureBootSmartCardDeployment)
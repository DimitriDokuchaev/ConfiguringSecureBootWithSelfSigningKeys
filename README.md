# ConfiguringSecureBootWithSelfSigningKeys
This tutorial will go through on how to configure your own private keys and add them to uefi allowing you to enable secure boot

## What is this?
This tutorial aims to facilitate the creation of your own private keys and certificates to sign efi binaries and kernels in order to have secure boot enabled on the machine, booting those binaries with your keys, but still retain the ability to boot microsoft windows with tthe MS keys already provisioned on the machine by defualt..

This tutorial is a balant rip off Sakaki's fantastic tutorial ["Sakaki's EFI Install Guide"](https://wiki.gentoo.org/wiki/User:Sakaki/Sakaki%27s_EFI_Install_Guide)

## What does this tutorial assume?
- you are doing this in linux release that does not rely on shim and mok manager
- this has been attempted successfuly in gentoo and arch
- I have never got this to work on ubuntu or debian (both rely on shim64.efi)

## The turotial

- pacman -S efitools sbsigntools
- mkdir -p -v /etc/efikeys
- chmod -v 700 /etc/efikeys
- cd /etc/efikeys

Let's dump the current keys:
- efi-readvar -v PK -o old_PK.esl
- efi-readvar -v KEK -o old_KEK.esl
- efi-readvar -v db -o old_db.esl
- efi-readvar -v dbx -o old_dbx.esl

Now that we dumped the original keys, let's create the new keys and certificates:
- openssl req -new -x509 -newkey rsa:2048 -subj "/CN=dimitri's platform key/" -keyout PK.key -out PK.crt -days 3650 -nodes -sha256
- openssl req -new -x509 -newkey rsa:2048 -subj "/CN=dimitri's key-exchange-key/" -keyout KEK.key -out KEK.crt -days 3650 -nodes -sha256
- openssl req -new -x509 -newkey rsa:2048 -subj "/CN=dimitri's kernel-signing key/" -keyout db.key -out db.crt -days 3650 -nodes -sha256

Let's set the correct permissions for the private keys:
- chmod -v 400 *.key

we will create a variety of files that may be used to update the keystore, We'll begin by creating a 'signed signature list' (aka '.auth') format version of the PK variable:
- cert-to-efi-sig-list -g "$(uuidgen)" PK.crt PK.esl
- sign-efi-sig-list -k PK.key -c PK.crt PK PK.esl PK.auth

We will do the same thing for the KEK:
- cert-to-efi-sig-list -g "$(uuidgen)" KEK.crt KEK.esl
- sign-efi-sig-list -a -k PK.key -c PK.crt KEK KEK.esl KEK.auth

Lastly for DB:
- cert-to-efi-sig-list -g "$(uuidgen)" db.crt db.esl
- sign-efi-sig-list -a -k KEK.key -c KEK.crt db db.esl db.auth

we actually do the same for the old DBXs:
- sign-efi-sig-list -k KEK.key -c KEK.crt dbx old_dbx.esl old_dbx.auth

Next, we'll create DER versions of each of our three new public keys, as follows:
- openssl x509 -outform DER -in PK.crt -out PK.cer
- openssl x509 -outform DER -in KEK.crt -out KEK.cer
- openssl x509 -outform DER -in db.crt -out db.cer

Let's create our compound keys (windows + our own keys)
- cat old_KEK.esl KEK.esl > compound_KEK.esl
- cat old_db.esl db.esl > compound_db.esl
- sign-efi-sig-list -k PK.key -c PK.crt KEK compound_KEK.esl compound_KEK.auth
- sign-efi-sig-list -k KEK.key -c KEK.crt db compound_db.esl compound_db.auth

Keys are generated, let's reboot and clear the keystore, once keystore is cleared time to provision the keystore with the new keys:
- cd /etc/efikeys
- efi-readvar
it should be empty

let's populate it:
- efi-updatevar -e -f old_dbx.esl dbx
- efi-updatevar -e -f compound_db.esl db
- efi-updatevar -e -f compound_KEK.esl KEK
- efi-updatevar -f PK.auth PK
- efi-readvar

you should see all the entries populated now.

Let's dump and keep the new keys on a safe place:
- efi-readvar -v PK -o new_PK.esl
- efi-readvar -v KEK -o new_KEK.esl
- efi-readvar -v db -o new_db.esl
- efi-readvar -v dbx -o new_dbx.esl

Now that everytthing is done we should now be able to sign both the kernel and the bootloader:
- sbsign --key /etc/efikeys/db.key --cert /etc/efikeys/db.crt --output /boot/vmlinuz-linux /boot/vmlinuz-linux
- sbsign --key /etc/efikeys/db.key --cert /etc/efikeys/db.crt --output /boot/efi/EFI/grub/grubx64.efi /boot/efi/EFI/grub/grubx64.efi

This is it.

## Acknowledgements
Thanks to:
- Томми on discord for hearing my rumbles and frustrations with secure boot on shim based OS.
- [Sakaki's awesome guide](https://wiki.gentoo.org/wiki/User:Sakaki/Sakaki%27s_EFI_Install_Guide) that goes extremely in depth on the creation of the keys (way more in depth than this guide)

## Bugs
I have tested the above procedure in a arch VM on qemu and everything works as expected, however on live hardware (DELL XPS 13 9360) somehow secure boot validates the bootloader (GRUB) but fails to validate the kernel, so it is actually able to launch an unsigned kernel.
This is an obvious problem, however the fact that the setup i am running is encrypted with luks and inside the luks containers there is a LVM, having that in place requires a password to actually decrypt the container and have access to GRUB which is signed (an unsigned version of GRUB wouldn't boot) to then launch the unsigned kernel, the attack surface is really limited.
I have tested this same exact procedure on gentoo and arch on MSI motherboard and secure boot works as expected (just like in the qemu vm) so it does validate the kernel, I am still trying to find why this happens on the DELL XPS 13 specifically 

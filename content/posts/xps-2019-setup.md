---
title: Dev laptop - Ubuntu on the XPS 13 9380
date: 2019-06-07T17:00:40Z
draft: false
author: 
 - Dave
summary: Getting my new XPS ready for doing dev work was frustrating at times but I'm happy with the outcome. Hopefully sharing my setup can help others.
---

_Updated: 27/06 with some additional notes on using fscrypt_

## Not cut out to be a Mac owner ##

Recently I started some contract software development work. As my ThinkPad x200
is getting a bit long in the tooth I was in the market for a laptop for work.
After working on Mac's for the past 5 years I initially bought a new MacBook Pro
13 inch (with touchbar) however the purchase experience all went a bit awry.
Firstly the delivery went missing and after an investigation from the delivery
firm I was informed "the driver no longer works for us, the package is
irrecoverable" Apple kicked off their own process. Customer service from Apple
was quite disappointing, it took them a good few days after the delivery company
told me the package was irrecoverable before they dispatched a replacement.

Secondly, Apple had in the meantime announced a refresh with improvements to the
MBP keyboard. After all those delivery issues and leaving me waiting for my
replacement I received an already old model. At this point I realised I'm happy
to be a Mac user but perhaps not cut out to be an owner.

Looking elsewhere I decided to return the MBP and ordered a new XPS 13: it's the
right form factor (albeit wrong aspect ratio), no longer has a nosecam and
_should_ run Linux well. They sell a version with Ubuntu preinstalled after all.

## Dell XPS 13 - 9380 ##

I went for a standard option with 1080p screen but upped the RAM to 16GB, this
was available with fast delivery. Models with Ubuntu preinstalled would have
taken a while longer to arrive. As I needed the laptop for work which had
already begun that wasn't really an option.

### Keyboard ###

The keyboard on the XPS 13 is nice, it has decent key travel and an escape key.

Unfortunately the keyboard I received had an issue where the left side corner of
the spacebar was a bit mushy and unresponsive. This appears to be [a common
issue][spacebar issue].

As the laptop came with a years premium support I phoned Dell and within 2
working days an engineer had visited me and replaced the keyboard. The issue is
no longer present \o/

### Installing Ubuntu ###

Once the laptop arrived I downloaded Ubuntu to a USB stick, rebooted to the
installer and couldn't see the hard drive, huh.. 🤔

Turns out the NVMe drive in the XPS ships in RAID compatible mode. I'm not sure
why but it has been an issue for people looking to install Linux on older models
[for a while now][]. The fix (toggling an option in the BIOS) breaks the
preinstalled Windows.

### Disk encryption ###

During the Ubuntu install I noticed the option to encrypt the home directory is
no longer available. This decision was taken in [ubuntu#1756840][ubuntu bug] -
I'm sure it wasn't taken lightly. If software which performs such an important
role can be described as buggy and under-maintained I'm pleased the Ubuntu team
took the decision to remove it.

In the past I've struggled a bit with LUKS and from what I've read about block
level encryption on NVMe drives I didn't fancy giving it a try. I don't want to
be left compiling my own kernels.

Once Ubuntu was installed I looked into alternative encryption options. Given
I'd also partitioned a single `/` partition [fscrypt][fscrypt] looked like the
best option:

 - Integrates with PAM to unlock files at login
 - It's an active project out of Google
 - It's written in Go so I'm more likely to understand its internals
 
 Once configured directory contents are encrypted this looks something like this
 
![Documents directory encrypted](/images/encrypted-documents.png)

These excellent [clear instructions on setting up fscrypt][fscrypt setup] worked
for me with a couple of caveats:

 - I didn't `rm -rf` the backup immediately! I left it around for a couple of
   reboots... just in case.
 - I had to reboot after enabling encryption on my ext4 device before
   configuring fscrypt.

I'm curious if we'll see fscrypt `/home/$USER` encryption as an option in future
Ubuntu installers.


### OUTSTANDING "Linux on the desktop" issues ###

It's still early days with the XPS 13 for me and I'm already following a couple
of issues on launchpad and waiting for fixes make their way upstream into the
kernel.

Missing bluetooth after sleep [ubuntu#1799988][missing bluetooth] - it looks
like a fix is on its way for this.

Occasionally I'm seeing a flicker on the screen. I've only noticed it on Ubuntu
since I switched to using Wayland but I haven't tried to debug further yet, the
flicker is brief and only slightly distracting. My reckon is that it's to do
with Intel Graphics on Wayland.

### Final thoughts ###

With all these issues considered I still have to say it's a great laptop and I'm
really happy with it.

Recently I've been using [Nix][nix] more (my super old x200 has NixOS on it) and
having [my home-configuration][home-config] in Nix and some nix-shell scripts
ready made setting up a new laptop for dev work much easier.

### Additional notes on fscrypt (27/06/2019)###

Some things which I didn't figure out immediately after setting up my laptop
with fscrypt have popped up and are worth noting. First I had an issue with
Firefox failing to download files, I was happily working around this and figured
it would be patched quickly. The second issue was with Docker volumes which gave
the game away. After rebooting my laptop and restarting some docker containers
which had persistent volumes I noticed errors. After opening a shell in the
container it was quick the volume was mounted while still encrypted using
fscrypt.

The version of fscrypt shipped in Linux kernel's below 5.1 has behaviour where
any attempts to move (e.g. using `mv`) unencrypted data into an encrypted folder
will fail. In a [recent commit][linux commit] the error returned from the kernel
changed. This tells tools (such as `mv`) to take a different action and instead
of renaming they copy the contents to a new file.

I highly recommend taking a read of that commit message as it's extremely well
written and helped me understand what was going on within the kernel. It has
also left me considering the power operating system's have with regards to how
we interact with data and ingraining habits (both good and bad) over time.
 
This Firefox [launchpad bug][launchpad issue] has confirmation that the issue
goes away when running a kernel >=5.1. I'm hoping for something similar with the
docker volume issue I've been seeing, but suspect that may be working a bit
differently.

[nosecam]: https://www.tomshardware.co.uk/dell-xps-13-price-specs-release-date,news-59718.html
[home-config]: https://github.com/davbo/cfg
[nix]: https://nixos.org/nix/
[missing bluetooth]: https://bugs.launchpad.net/ubuntu/+source/linux/+bug/1799988
[spacebar issue]: https://www.google.com/search?q=dell+xps+keyboard+spacebar+left+corner
[ubuntu bug]: https://bugs.launchpad.net/ubuntu/+source/ecryptfs-utils/+bug/1756840
[for a while now]: https://askubuntu.com/questions/696413/ubuntu-installer-cant-find-any-disk-on-dell-xps-13-9350
[fscrypt]: https://github.com/google/fscrypt
[fscrypt setup]: https://tlbdk.github.io/ubuntu/2018/10/22/fscrypt.html
[rename syscall]: http://man7.org/linux/man-pages/man2/rename.2.html
[linux commit]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f5e55e777cc93eae
[launchpad issue]: https://bugs.launchpad.net/ubuntu/+source/firefox/+bug/1796661

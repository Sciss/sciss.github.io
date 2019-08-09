---
layout: post
title: "Updating Debian from 9 (Stretch) to 10 (Buster)"
description: ""
# category: ""
tags: [linux]
---
{% include JB/setup %}

I am less adventurous than some of my colleagues who are running Debian unstable as their main operating system. I started using Debian when version 8 (Jessie) was in freeze,
so I almost started with a stable Jessie. When version 9 (Stretch) came out, I just updated my apt sources, and the whole process worked quite nicely. Since a few weeks,
version 10 (Buster) was marked stable, and some of the recent GPG key server attacks prompted me to perform this update, since gpg 2.1.18 was giving me trouble, and the
newer version 2.2.12 promised relief. In this post, I want to summarise the hiccups I had to solve, which are probably particular to me coming from a system that was
originally originally Jessie before its release.

You can basically follow [this official instructions](https://www.debian.org/releases/buster/mips/release-notes/ch-upgrading.en.html) for upgrading. Looking through the
[Issues to be aware of chapter](https://www.debian.org/releases/buster/mips/release-notes/ch-information.en.html), I took note of section 5.1.5 legacy network names.
`70-persistent-net.rule` was the only file mentioning `eth0`, so it boiled down to commenting out the line in that file and rebooting.

## Running the apt upgrades

Next I uninstalled stuff I had from other sources, as a precaution. `aptitude search '~i(!~ODebian)'` gives you hints what those packages are (anything not marked 'A' was
installed manually and not automatically). I then ran

    $ apt update

I still had some issues, like:

    The following packages have unmet dependencies:
     gnustep-base-runtime : Depends: gnustep-base-common (= 1.26.0-4) but 1.24.9-3.1 is to be installed
     libgnustep-base1.26 : Depends: gnustep-base-common (= 1.26.0-4) but 1.24.9-3.1 is to be installed
    E: Broken packages

Those were part of the non-automatically installed packages, so step by step I could remove offending packages.
When unsure, you can check a suspicious package with `apt show package-name`. 

Finally, I removed stuff like Skype, node.js and VS Code, along with their custom apt sources files, leaving me just with one
sources file needing update:

    $ sudo sed -i 's/stretch/buster/g' /etc/apt/sources.list

(I stole that line [from this tutorial](https://linuxconfig.org/how-to-upgrade-debian-9-stretch-to-debian-10-buster)).
Having backed up all important data to an external harddisk (including invisible dot-files from the home directory), I then ran

    $ apt update
    $ sudo apt upgrade

This worked without major conflicts. Some dialog popped about about 'minissdpd', I just confirmed the default options.
The final `update-initramfs`, which recreats your bootloader stuff, used kernel `4.19.0-5-amd64`, but
gave me some warnings like

    W: Possible missing firmware /lib/firmware/i915/bxt_dmc_ver1_07.bin for module i915
    W: Possible missing firmware /lib/firmware/i915/skl_dmc_ver1_27.bin for module i915
    ...

I searched on the net, and preemptively ran this

    $ sudo apt install firmware-linux

(FYI, I have included `non-free` in my sources, this might be relevant). The warnings disappeared, hurrah! I rebooted, and finally:

    $ sudo apt full-upgrade

No surprises, no problems. Finally, I took some time to review again with `aptitude search` if there were old packages no longer need, removed them, and performed
some autoremove and purge operations.

## Hiccups

So the basic process was quite smooth. Most settings seemed to have survived. I encountered two/three problems though, the solution of which I want to document here:

### git has SSH problems

I could not push git repositories, because there was some SSL problem, giving me a line like

    sign_and_send_pubkey: signing failed: agent refused operation

It turned out to be related to permissions (I think they were too permissive, perhaps
because I originally copied keys from a previous computer). This was solved by running

    $ chmod 600 .ssh/id_rsa

## ssh, rsync, sftp fail to connect to my web space

I noticed that I can no longer connect to my web space provider using sftp in Nautilus, then noticed that this is a general SSH problem, as I also couldn't
log into the server using `ssh`. The error was

    ssh_dispatch_run_fatal: Connection to 81.169.145.126 port 22: incorrect signature

It seems that the known hosts cache is somehow corrupt. I probably should wipe it completely, for now I just removed the keys relevant to this server.
It's important to remove both hostname and IP, otherwise you get something like

    Warning: the ECDSA host key for 'ssh.strato.de' differs from the key for the IP address '192.168.1.123'

So the clean-up steps were:

    $ ssh-keygen -R ssh.strato.de
    $ ssh-keygen -R 81.169.145.126

### Custom key mappings are gone

A while ago my left alt and left shift keys starting to cause trouble, eventually ceasing to work altogether. I had thus swapped left alt with left
super ("windows key"), and changed caps lock to work as the left shift key. The former mapping was preserved, while the latter was gone.
The reason was that I was using `xmodmap` to set the behaviour, but Debian 10 no longer uses Xorg but rather Wayland for its windowing system.
Don't ask me what that means, but the consequence is that `xmodmap` became meaningless. To cut it short,
[this blog post](https://www.beatworm.co.uk/blog/keyboards/gnome-wayland-xkb) proved enormously useful, essentially identifying and fixing the
problem.

What you do, is add and edit some stuff in `/usr/share/X11/xkb`, and then chose a custom rule in `dconf`:

First, create a new file `/usr/share/X11/xkb/symbols/mycapslock` with a contents similar to the `ctrl_modifier` rule already defined in the system:

```
// This changes the <CAPS> key to become a Shift modifier,
// but it will still produce the Caps_Lock keysym.
hidden partial modifier_keys
xkb_symbols "shift_modifier" {
    replace key <CAPS> {
        type[Group1] = "ONE_LEVEL",
        symbols[Group1] = [ Caps_Lock ],
        actions[Group1] = [ SetMods(modifiers=Shift) ]
    };
    modifier_map Shift { <CAPS> };
};
```

Then this 'overlay' must be included as a rule in `/usr/share/X11/xkb/rules/evdev`. Find the line that specifies `ctrl_modifier` and a new line
below:

    caps:shift_modifier = +hhcapslock(shift_modifier)

Similarly, edit `/usr/share/X11/xkb/rules/evdev.lst` and add the new rule in a new line under the `ctrl_modifier` rule:

    caps:shift_modifier Caps Lock is also a Shift

Theoretically you should also edit `evdev.xml` which is what the GNOME Tweak application will present to you. I didn't bother, and so
I can't see that option there. But I can select the rule now in `dconf`. Open it and go to section 
`/org/gnome/desktop/input-sources/xkb-options`. Here, untick 'Use default value', and enter custom value `['caps:shift_modifier']`.
Apply and quit. Now it should work. I had to re-enable _Keyboard \& Mouse_ > _Additional Layout Options_ > _Alt/Win key behavior_ > 
_Left Alt is swapped with Left Win_ in Tweak after this. Now both modification seem to be in place, fingers crossed
that they'll stay there.

-----

I hope this information is useful to anyone running into similar issues.

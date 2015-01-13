---
layout: post
title: "Almost One Year with Linux Audio"
description: ""
# category: ""
tags: [linux, audio]
---
{% include JB/setup %}

Having had to return my previous laptop to the University of Plymouth after finishing my PhD, my new work place, the IEM in Graz, thankfully provided me with a new one. This was about the same spec, a MacBook Pro early 2010 model. Given that I had to re-install all my data from scratch, and given that quite a few of my colleagues at the IEM embrace Linux and FLOSS, I decided to jump into the cold water and switch from OS X to Linux. The following paragraphs describe my experience with this system that I have chosen almost a year ago.

## Configuring the System

I went for _Debian_, because this is the distribution most of colleagues use, so if I run into trouble I could ask them. I would recommend having someone around that can help especially in the first few weeks, because even if you are tech savvy (like I consider myself I am) you will face quite a few problems in the beginning. Some of them because the system is in beta (I went for Debian 8 aka _Jessie_ which although now in "freeze" was still a moving target in March 2014), some of them simply because you wouldn't know where the switches are.

I also decided for _Gnome 3_, something that hardcore Linux geeks probably dislike, because it's an end-user friendly desktop environment, actually quite pretty and not too dissimilar from OS X. In fact, it took me only two or three weeks to feel entirely comfortable in Gnome, and now when I have to go back to an OS X machine, I find that system limiting and I struggle to remember the shortcuts.

## Installing Software

If you install the Gnome meta-package (a package that bundles other packages), you already get a great desktop system with all the standard tools, from _Iceweasel_ (a rebranded _Firefox_) to _Gimp_ (replaces _PhotoShop_) to _LibreOffice_ (replaces _Word_, _Excel_, _PowerPoint_), and a lot of small utilities that replace all the OS X stuff, like _Sound Juicer_ (Audio CD import and playback), _Brasero_ (burning DVDs), _Nautilus_ aka 'Files' (_Finder_), _Gedit_ (_TextEdit_), _Evince_ aka 'Document Viewer' and _EyeOfGnome_ aka 'Image Viewer' (_Preview_), _Shotwell_ (_iPhoto_), _Cheese_ (_Photo Booth_), system monitor, terminal etc. I didn't want to go with the bundled e-mail client and calendar, so I installed _Icedove_ (rebranded _Thunderbird_) with a calendar plug-in.

The software installation process using the _apt_ tool is incredible simple and powerful. In the beginning I used the graphical Gnome tool _gdebi_, but I strongly recommend forgetting about this. It has a lot of problems, the shell tool apt is a magnitude better. Since I use a lot of Java and Scala based software, I installed _OpenJDK 6_ and _OpenJDK 7_ via apt. I also installed _SuperCollider_, _QJackCtl_, _Audacity_, _VLC_, _Inkscape_, _Texmaker_ and quite a few other tools this way.

The JVM world is kind of independent of apt then. For example, my preferred development environment, _IntelliJ IDEA_, is just a download that you can place anywhere. All my own software, written in _Scala_, is cloned from their respective Git repositories, or downloaded with _sbt_ from a _Maven_ repository.

## Woes

Especially in the beginning, Debian Jessie was far from stable. I had to learn the hard way that once most of your stuff is running, it's perhaps a good idea not to run `apt-get upgrade` whenever possible. Among the problems I encountered were entire kernels that wouldn't boot but just produce graphics garbage, network manager problems, power manager problems where the laptop didn't wake up from sleep, problems with the graphics card driver. I can say that since September 2014 I haven't encountered any new surprises. The screen shot tool occasionally decides to screw up the graphics system to the point that I can only reboot, but that's avoidable.

Two big woes remained. First, getting an external video projector to work on the mini display port is a lottery. It seems to work with a direct VGA or DVI cable to the projector, but as soon as the cable gets very long or a KVM switch sits inbetween, the graphics driver simply believes no external device is connected and refuses to output to the second display. Needless to say, this is a bummer for teaching and presenting, especially if you are going to a venue you don't know.

The second issue was the hardware support for audio...

## Audio Support - Software

Linux has a bit of a complicated story in terms of audio driver standards. With the standard Gnome, the contemporary system is _Pulse Audio_, whereas for pro audio applications you will want to use _Jack_. You can google dozens of pages that explain how the two are mutually exclusive and how you can fiddle around to route one through the other. I must admit, I hadn't had the patience to understand all the details, and you will also learn that the Internet is loaded with "useful" pages about Linux, the problem being often that these tips work only for a specific distribution or versions of systems, often being "obsolete" and so on. I ended up leaving everything in place. When I run a pro audio application, I start QJackCtl and Pulse Audio is blocked. When I need to hear a stream in a web browser again or make a Skype call, I "kill" Jack and Pulse Audio comes back by itself. Not ideal, but it has worked for me so far.

Jack and QJackCtl are amazing pieces of software. They look inconspicuous, but once you've learned how to program the automatically activated patching matrices, or once you've routed multi-channel audio between two computers using _NetJack_, you understand the power behind it. For example, many of my applications are SuperCollider based, and I can pass a different Jack client name to each of them. They will then appear as different devices in Jack and so I can have individual I/O patches for each of them.

## Audio Support - Hardware

The internal sound card of the Mac works without problem. The Linux driver seems to be a bit more rock n'roll in terms of the output levels it can provide, so the other day when I was setting up an exhibition and had a YouTube channel running a few metres away in the background, the play list contained a very loud video, the result of which was that my left laptop speaker got destroyed (physically!). Ouch...

I'm the proud owner of a second-hand _MOTU 828 mkII_, this has a few years with me, so I had hoped that I can just stick to using it for concerts. There is an ambitious project, _FFADO_, to provide support for Firewire audio on Linux, but unfortunately the 828 mkII was giving me a hard time. Sometimes it worked, but after an indefinite amount of time, often just minutes, the device would start producing random beeps across random output channels; effectively rendering it useless under Linux.

You will then go and investigate alternative devices. And you will find that indeed not a single company officially supports Linux. Ouch... There is again dozens of pages on the Internet that will tell you what works and what not, but in the end none of this is reliable information. You will have to try the hardware yourself.

Apparently _RME_ is a good candidate. The _Hammerfall_ based cards work really well under Linux, but of course the problem is that Apple at some point decided to drop the card slot of the laptops, and now we're stuck with Firewire, USB, or even Thunderbolt if you are lucky and own a newer computer. For my last exhibition, we had a PC laptop with express card slot, and that beast could run 64 channels from an RME _Madiface_---one great piece of equipment if you can afford the whole infrastructure with MADI-to-ADAT and ADAT-to-analog. But I need a device for my personal work, to run concerts with at least 8 analog inputs and 8 analog outputs. Better if I can have an additional ADAT in, so I can still make use of my precious RME _OctaMic-D_.

I have given up on Firewire, having tried all sorts of gear from my lab, like a _Fireface 400_ and _Fireface 800_. Nothing really worked. So that left me with USB. The good news is there is such thing as a 'class compliant' USB 2.0 mode. This basically means that your card is (or should be) automatically supported by your system without a dedicated driver. We have a _Fireface UCX_ in the lab, and this for example works great on Linux via USB (although I haven't had time to test the ADAT I/O). Unfortunately it was a bit over my budget, and the cheaper _Fireface UC_ does not support this CC mode.

After a bit of research, I kept hearing that _Focusrite_ is quite Linux friendly, and so with my I/O spec this put the rather new _Scarlett 18i20_ into the focus. This thing costs 60% of the UC and 40% of the UCX, while giving you ten instead of eight analog outputs (not talking about the stupid idea to put outputs 7+8 on one stereo jack on the RME).

## Focusrite Scarlett 18i20 - First Impressions

Today the Scarlett arrived. It will take some time to battle test this box, so the following impressions have to be taken cautiously. First of all, while class-compliant mode means the box should work directly, it doesn't mean that you have access to all of the functionality. In particular you need a special software to configure the mixer matrix. Apparently, there is a patch for the ALSA Linux architecture that allows one to access this matrix directly from Linux. I haven't had time to fiddle around with this patch yet, so the easiest solution is to connect the box once to a computer running OS X or Windows.

For that matter, I installed the _Scarlett Mix Control_ software (v1.8 as of this writing) on my partner's MacBook running OS X 10.7 (a horrible OS X version by the way, but OS X 10.6, still present on a small partition of my own laptop, is not supported by Focusrite). First time you open it, it asks you to update the firmware. This only took a second and didn't cause any problem.

I changed the rather odd default matrix that is just putting some stereo mix on a few of the outputs, for a direct 1:1 wiring of the 20 DAW outputs to the 20 hardware outputs of the Scarlett (I skipped the SPDIF and actually just connected 8 analog outputs and 8 ADAT outputs). The same with the inputs. I selected "Save to hardware", disconnected the box and finally got back to my Linux machine.

All 18 inputs and 20 outputs appear directly in Jack. I haven't tested the ADAT output yet, but the analog I/O works all fine, and I also managed to record multiple microphone channels through the OctaMic connected to the ADAT input of the Scarlett (the Scarlett is clock master, giving the word-clock to the OctaMic).

So far I noticed rather regular XRUNs when the Wifi is enabled, but this seems to be a common problem. While the Wifi is off, I haven't had any issues with XRUNs yet. Also note that I'm too lazy to fight with low-latency or realtime kernels, so right now I'm running the off-the-shelf Debian Jessie (kernel 3.16). Things can probably get better if you invest more energy.

![Scarlett Setup Photo](/images/scarlett_5825.jpg)

![Scarlett Setup Photo](/images/scarlett_5826.jpg)

### Conclusion and Next Steps

So at day 1, the experience with the Focusrite Scarlett 18i20 has been smooth. From the overall impression of the box, I must say it looks quite pretty with the red metal. For travelling, the box is rather large (19" wide and around 25 cm deep) and heavy, especially compared to a Fireface 400 or Fireface UC. The knobs make a quite cheap impression, especially the monitor and headphones knobs, while the input gain knobs feel a bit more robust. The same goes for the phantom power, mute and dim switches, I guess this is where they tried to save the money. On the display, I am especially missing an LED to indicate the ADAT input status, I would guess it can get tricky if you are in a situation where the ADAT input signal is not showing up in the software and you don't know whether the box sees it at all.

On the positive side, having two headphones outputs is quite cool, even though they are hard-wired to the analog outputs 7 to 10 (plus the extra gain control). I haven't tested the quality of the microphone preamps yet (this is a minor issue since I have the OctaMic), and also I haven't tried to use the MIDI ports yet.

I will try to battle-test the Scarlett very soon in a rehearsal with multi-channel I/O setup, and then I will publish another (hopefully positive) blog entry.

## Last Words - For Now

After almost one year with Linux, despite the mentioned frustrations, I must say I don't regret having made the switch from OS X. The operation system in my opinion is absolutely superior, especially if you develop your own programs and use a lot of open source software. The sore point remains pro audio driver support, but with these new class compliant USB interfaces it appears that you can work professionally in a multi-channel setting, even if USB perhaps is not the dream low latency solution.


---
layout: post
title: Build a wayland desktop by yocto
category: linux
tag: Embedded and Linux
comments: 1
---

Wayland has been introduced as a replacement to the x11 for a long time but there is still a long way
to go in the pc linux world.

I believe wayland will be widely used in embedded platform soon, because things like xserver use too many CPU time to do copy things, and ARM core are weak to do memory-copy things, compare to X86, which make a really bad performance.  

Wayland have a modern desgin which make full use of GPU, so it have a more better performance than X11 in embedded platform.
We should drop xserver now! (and fucking qte, minigui).

# build

First, we should have a yocto sdk installed, and I use rockchip yocto sdk.  
http://opensource.rock-chips.com/wiki_Yocto

	repo init -u https://github.com/rockchip-linux/manifests -b yocto -m master.xml
	repo sync
	DISTRO=rk-wayland . ./setup-environment

Then, we need clone [“meta-qt5-extra”](https://layers.openembedded.org/layerindex/branch/master/layer/meta-qt5-extra/).

	git clone https://github.com/schnitzeltony/meta-qt5-extra.git

And, include meta-qt5-extra and meta-gnome in bblayer.conf.

	${BSPDIR}/sources/meta-openembedded/meta-gnome \
	${BSPDIR}/sources/meta-openembedded/meta-initramfs \
    ${BSPDIR}/sources/meta-qt5-extra \

Add DISTRO_FEATURES_append += " wayland pam x11" to local.onf.

Now, append `liri-world` to rk-image-machine-test and build it.

 	bitbake rk-image-machine-test

HAHA, after flash the rootfs created by yocto, I get a wayland based modern design desktop on rockchip platform!
Faster than xserver!



Liri is based on Qt, which use gstreamer for multimedia, so the mediaplayer in Liri use hardware decode on rockchip platform by default.



# link

https://wiki.archlinux.org/index.php/Liri  

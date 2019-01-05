---
title: How I taught Mac OS X to write into an NTFS filesystem.
date: 2015-05-20 00:00:00 +05:30
categories:
- Revolt
- Technology
- Apple
tags:
- Mac OS X
- Yosemite
- NTFS
- Homebrew
- FUSE
layout: post
description: Workaround to make Mac OS X write into an NTFS filesystem.
modified: 
image:
  feature: Revolt/Default.png
  credit: JLJ
  creditlink: http://www.swiftjuggler.com
comments: true
share: true
---

I was really having a hard time trying to understand why Mac OS X was not writing to most of my external drives, well I said most because I had some exceptions with some of my flash drives. Funny part is that I almost thought that I didn't have administrative privileges to write into external drives and even thought it must be a bug in some complicated specification of some of those special flash drives. Mostly I only wanted to transfer some mega bytes of data which I usually shared via dropbox or such easier medium, but yesterday I happened to download a full coursework from iTunesU which was 15GB in size and wanted to share it with my friend through his external HDD

I felt so stupid when I realized that it was neither a bug nor any admin privilege issues, rather it was just differences in the filesystems of  those external drives. Those special flash drives happened to be formatted using  FAT32 and the rest was NTFS, apparently Mac OS X doesn’t support writing into NTFS out of the box. I searched for a solution and found this [GitHub Gist of bjorgvino](https://gist.github.com/bjorgvino/f24e5c079b92f921b765) which really helped and I thought I'll share it.

Any work around in both Linux and Mac OS X always starts with a Terminal, so open it up and get ready to spit out these commands.

1. If you do not have [Homebrew](http://brew.sh/), install it! `ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"`
2. Now lets install `osxfuse`, now if you previously had installed both Homebrew and `osxfuse `, chances are that it might have unsigned kexts which are banned now. So you might have to uninstall `osxfuse` with `brew cask uninstall osxfuse`
   * So for the rest of the people out there like me, lets update homebrew using `brew update`
   * Now we have to install Homebrew for binaries called the "cask", `brew install caskroom/cask/brew-cask`
   * `osxfuse` comes as a binary package and we install it using cask `brew cask install osxfuse`
3. `ntfs-3g` is actually a driver that can enable your Mac OS X to R/W into NTFS, install it using `brew install ntfs-3g`
4. Now to make the system work seamlessly, we have to make drives mount automatically. Create a symlink for `mount_ntfs`
   * `sudo mv /sbin/mount_ntfs /sbin/mount_ntfs.original`
   * `sudo ln -s /usr/local/Cellar/ntfs-3g/2014.2.15/sbin/mount_ntfs /sbin/mount_ntfs`
5. Type `ls -l /sbin/mount_ntfs*` and see if the following appears as the output
   * `/sbin/mount_ntfs -> /usr/local/Cellar/ntfs-3g/2014.2.15/sbin/mount_ntfs`
   * `/sbin/mount_ntfs.original -> /usr/local/Cellar/ntfs-3g/2014.2.15/sbin/mount_ntfs`
6. Reboot the system and you are good to go!

So yeah that’s how I taught Mac OS X to write into NTFS filesystem.

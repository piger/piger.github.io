---
title: "Into the Breach on OS X with Wine"
date: 2018-02-28
summary: Quick and simple tutorial on how to run "Into the Breach" on OS X using Wine.
tags:
- macOS
- games
- wine
thumbnail: thumbnails/ITB_ss_1.png
categories:
- Games and Videogames
---

Quick tutorial blah.

![Thumbnail](/thumbnails/ITB_ss_1.png)

## Requirements

- [Wineskin](http://wineskin.urgesoftware.com/)

That's right. You only need to download Wineskin.

## Instructions

- Download and install Wineskin
- Run Wineskin, download the latest engine (I've used **WS9Wine2.22**)
- Select "Create New Blank Wrapper"
- When it finishes creating the wrapper, right click on the wrapper icon and select "Show Contents"
- Double click on **Wineskin** (note: this is the Wineskin configuration tool; it's different from
  the "main" Wineskin Winery executable)
- Select "Advanced", "Tools", "Winetricks"
- Search "steam" and install it
- When it finishes installing Steam, click "Test Run" in the "Tools" window of Wineskin
- Let Steam complete its installation by downloading all the updates
- Exit Steam when the update is completed; do not run Steam after the installation is completed.
- Go back to Finder, go to the window containing your wrapper (i.e. the directory that *contains*
  the Wineskin configuration tool); double click on your wrapper to launch it.
- Log In into Steam
- Install Into the Breach
- Enjoy :)

Or check this more detailed guide on [reddit](https://www.reddit.com/r/IntoTheBreach/comments/80iqjr/for_the_macs/).

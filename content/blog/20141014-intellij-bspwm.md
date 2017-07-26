---
title: "Using IntelliJ IDEA / Android Studio with bspwm"
date: 2014-10-14T13:32:54+02:00
slug: "using-intellij-idea-android-studio-with-bspwm"
---

I decided to give [IntelliJ IDEA](https://www.jetbrains.com/idea/) a try for a new Android project in my also newly configured window manager... And after the splash screen I could only see a transparent frame.

After a bit of research it turns out that some Java apps don't get along well with some [non-reparenting window managers](http://en.wikipedia.org/wiki/Re-parenting_window_manager) like [bspwm](https://github.com/baskerville/bspwm) because they are not acknowledged as such... It can be fixed by setting the window manager name property to one that is known by the JVM, by using [wmname](http://tools.suckless.org/wmname) like follows:

	$ wmname LG3D

After this change IDEA works flawlessly! Well, except maybe for the font anti-aliasing, but that's another story...

[Source and more info](http://awesome.naquadah.org/wiki/Problems_with_Java)

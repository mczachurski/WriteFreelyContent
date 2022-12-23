# macOS Mojave dynamic wallpapers (II)

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0062.png)

> This is the article created at Jun 30, 2018 and moved from Medium.

In my [previous article](https://itnext.io/macos-mojave-dynamic-wallpaper-fd26b0698223) I described how dynamic wallpapers works. I didn’t know then what some of the properties in metadata means. I asked if somebody else knows what that properties means, and I’ve got a response really quickly. On Twitter @zwaldowski wrote to me explanation what all properties stands for.
<!--more-->

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0063.png)

Thus, as I wrote previously, in HEIC file we have metadata which looks like on below snippet.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
 <key>si</key>
 <array>
  <dict>
   <key>a</key>
   <real>-0.34275283875350282</real>
   <key>i</key>
   <integer>0</integer>
   <key>o</key>
   <integer>1</integer>
   <key>z</key>
   <real>270.9334057827345</real>
  </dict>
    ...
 </array>
</dict>
```

We have 16 `dict` elements (one for each image) with four keys:

- a — altitude
- i — image index
- o — indicates in which desktop theme image should be displayed. 0 — displays in both mode (light/dark). 1 — displays only in light mode.
- z — azimuth

Thanks to [altitude and azimuth](https://en.wikipedia.org/wiki/Horizontal_coordinate_system) w exactly know where the Sun was when image was taken.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0064.png)

This idea is brilliant, because thanks to this information macOS can change images differently during Summer and during Winter. System knows where the Sun is and it will choose image which was taken in similar conditions. **Brilliant**.

---

Based on that knowledge I prepared new dynamic wallpaper. Here is the video how it looks like:

You can download that wallpaper from my Dropbox: [https://www.dropbox.com/s/kd2g59qswchsd0v/Earth%20View.heic?dl=0](https://www.dropbox.com/s/kd2g59qswchsd0v/Earth%20View.heic?dl=0)

Thank you for your help and feedback!

---

**Update:** I created simple console application for macOS which can help you with creating custom dynamic wallpapers: [https://github.com/mczachurski/wallpapper](https://github.com/mczachurski/wallpapper)
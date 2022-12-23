# macOS Mojave wallpaper (III)

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0065.png)

> This is the article created at Jul 31, 2018 and moved from Medium.

In my previous articles ([part 1](https://writefreely.social/mczachurski/macos-mojave-dynamic-wallpaper), [part 2](https://writefreely.social/mczachurski/macos-mojave-dynamic-wallpapers-ii)) I described how Apple built dynamic wallpapers in macOS 10.14. In the latest macOS beta (beta 5) Apple introduced some small improvements in that area. First of all you can notice changes in desktop settings screen.
<!--more-->

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0066.png)

Now when we chose dynamic wallpaper we can choose how system will display it. We have three options: **dynamic** (wallpaper will change during whole day), **light** (system will display only one, light picture) and **dark** (system will display only one, dark picture).

For that purpose Apple changed also `plist` metadata in `HEIC` file in dynamic wallpaper. Now Mojave wallpaper contains metadata like on below snippet:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>ap</key>
	<dict>
		<key>d</key>
		<integer>15</integer>
		<key>l</key>
		<integer>0</integer>
	</dict>
	<key>si</key>
	<array>
		<dict>
			<key>a</key>
			<real>-0.34275283875350282</real>
			<key>i</key>
			<integer>0</integer>
			<key>z</key>
			<real>270.9334057827345</real>
		</dict>
                     ...
		<dict>
			<key>a</key>
			<real>-38.04743388682423</real>
			<key>i</key>
			<integer>15</integer>
			<key>z</key>
			<real>53.509085812513092</real>
		</dict>
	</array>
</dict>
</plist>
```

Thus Apple removed `o` parameter from `si` array. We have also new section: `ap` (this section is optional). In that section we have two parameters:

- `l` — index of picture for light static wallpaper
- `d` — index of picture for dark static wallpaper

Also Apple introduced new dynamic wallpaper: _Solar gradients:_

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0067.png)

---

If you want to prepare your own dynamic wallpaper you can use [this](https://github.com/mczachurski/wallpapper) console application from GitHub.

---

Unfortunately still in macOS we have some bugs on desktop settings screen.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0068.png)

Dynamic wallpaper is not rendered properly on preview boxes, and there is some issue on drop down list (it seems that we have here two different drop downs).

Let’s hope that Apple will fix this in the final version of macOS Mojave. I also encourage artists to create wonderful dynamic wallpapers!
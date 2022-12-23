# macOS Mojave dynamic wallpaper

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0057.png)

> This is the article created at Jun 29, 2018 and moved from Medium.

How Apple built dynamic wallpapers? And is it possible to create your own dynamic wallpaper for macOS? I spent some time because I would like to answer to the both above questions.
<!--more-->

---

In Mojave we can choose new type of wallpaper: dynamic.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0058.png)

Depending on day time, system displays different wallpaper. At night it is picture of Mojave at night, during day it’s picture of Mojave taken during day. So at night we have dark wallpaper at day we have light wallpaper.

All built in wallpapers in macOS we can find in folder: `/Library/Desktop Pictures`. Also here we can find dynamic wallpaper.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0059.png)

Dynamic wallpaper is saved as a HEIC file (`Mojave (Dynamic).heic`). More information about this type of file you can find on [Wikipedia](https://en.wikipedia.org/wiki/High_Efficiency_Image_File_Format). Generally this is type of file which Apple uses in iOS devices. Photos in iOS (also live photos) are stored as a HEIC file. That kind of file can contains multiple images/thumbnails and metadata in a single file.

There is really good online tool where we can check content of Mojave wallpaper file: [https://strukturag.github.io/libheif/](https://strukturag.github.io/libheif/). You can upload `Mojave (Dynamic).heic` file and you will see that in this single file there is 16 separate pictures taken in different day phase.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0060.png)

So maybe it’s enough to create new HEIC file with 16 separate images and macOS will show them properly. Let’s try it!

First I had to prepare 16 different images.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0061.png)

I know, I know — they are not impressive :-). However for our experiment they will be enough.

Now I have to prepare application which convert set of images to one HEIC file. I prepared simple console application in Swift.

```swift
import Foundation
import AppKit
import AVFoundation

extension NSImage {
    @objc var CGImage: CGImage? {
        get {
            guard let imageData = self.tiffRepresentation else { return nil }
            guard let sourceData = CGImageSourceCreateWithData(imageData as CFData, nil) else { return nil }
            return CGImageSourceCreateImageAtIndex(sourceData, 0, nil)
        }
    }
}

let output = "wallpapers-new/output.heic"
let quality = 0.9
var imageData: Data? = nil

if let dir = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask).first {

    let destinationData = NSMutableData()
    if let destination = CGImageDestinationCreateWithData(destinationData, AVFileType.heic as CFString, 16, nil) {
        let options = [kCGImageDestinationLossyCompressionQuality: quality]

        for index in 1...16 {
            let fileURL = dir.appendingPathComponent("wallpapers-new/\(index).png")
            let orginalImage = NSImage(contentsOf: fileURL)

            if let cgImage = orginalImage?.CGImage {
                    CGImageDestinationAddImage(destination, cgImage, options as CFDictionary)
            }
        }

        CGImageDestinationFinalize(destination)
        imageData = destinationData as Data

        let outputURL = dir.appendingPathComponent(output)
        try! imageData?.write(to: outputURL)
    }
}
```

Application requires that in my `Documents` folder I have `wallpaper-new` folder with 16 images (from `1.png` to `16.png`). I can run application and after that I have new `output.heic` file in the same directory.

After setting this file as a wallpaper I had only one image displayed during whole day. So it seems that this file have to contains something extra. I decided to check what metadata is inside original Mojave wallpaper. I downloaded [libheif](https://github.com/strukturag/libheif) library (I modified a little bit `heif-info` application) and I’ve got below result:

```
Number of top level images: 16
    image #0
        id: 61
        width: 5120
        height: 2880
        is primary: yes
        number of metadata: 1
        metadata type: mime
        content type: application/rdf+xml
        metadata size: 3302
        metadata content:
<?xpacket begin="" id="W5M0MpCehiHzreSzNTczkc9d"?> <x:xmpmeta xmlns:x="adobe:ns:meta/" x:xmptk="XMP Core 5.4.0"> <rdf:RDF xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#"> <rdf:Description rdf:about="" xmlns:apple_desktop="http://ns.apple.com/namespace/1.0/" apple_desktop:solar="YnBsaXN0MDDRAQJSc2mvEBADDBAUGBwgJCgsMDQ4PEFF1AQFBgcICQoLUWlRelFhUW8QACNAcO7vOubr3yO/1e+pmkOtXBAB1AQFBgcNDg8LEAEjQFRxqCKOFiAjwCR6waUkDgHUBAUGBxESEwsQAiNAVZV4BI4c+CPAEP2uFrMcrdQEBQYHFRYXCxADI0BWtALKmrjwIz/2ObLnx6l21AQFBgcZGhsLEAQjQFfTrJlEjnwjQByrLle1Q0rUBAUGBx0eHwsQBSNAWPrrmI0ISCNAKiwhpSRpc9QEBQYHISIjCxAGI0BgJff9KDpyI0BENTOsilht1AQFBgclJicLEAcjQGbHdYIVQKojQEq3fAg86lXUBAUGBykqKwsQCCNAbTGmpC2YRiNAQ2WFOZGjntQEBQYHLS4vCxAJI0BwXfII2B+SI0AmLcjfuC7g1AQFBgcxMjMLEAojQHCnF6YrsxcjQBS9AVBLTq3UBAUGBzU2NwsQCyNAcTcSnimmjCPAGP5E0ASXJtQEBQYHOTo7CxAMI0BxgSADjxK2I8AoalieOTyE1AQFBgc9Pj9AEA0jQHNWsnnMcWIjwEO+oq1pXr8QANQEBQYHQkNEQBAOI0ABZpkFpAcAI8BKYGg/VvMf1AQFBgdGR0hAEA8jQErBKblRzPgjwEMGElBIUO0ACAALAA4AIQAqACwALgAwADIANAA9AEYASABRAFMAXABlAG4AcAB5AIIAiwCNAJYAnwCoAKoAswC8AMUAxwDQANkA4gDkAO0A9gD/AQEBCgETARwBHgEnATABOQE7AUQBTQFWAVgBYQFqAXMBdQF+AYcBkAGSAZsBpAGtAa8BuAHBAcMBzAHOAdcB4AHpAesB9AAAAAAAAAIBAAAAAAAAAEkAAAAAAAAAAAAAAAAAAAH9"/> </rdf:RDF> </x:xmpmeta><?xpacket end="w"?>
    image #1
        id: 123
        width: 5120
        height: 2880
    image #2
        id: 184
        width: 5120
        height: 2880
    image #3
        id: 245
        width: 5120
        height: 2880
    image #4
        id: 306
        width: 5120
        height: 2880
    image #5
        id: 367
        width: 5120
        height: 2880
    image #6
        id: 428
        width: 5120
        height: 2880
    image #7
        id: 489
        width: 5120
        height: 2880
    image #8
        id: 550
        width: 5120
        height: 2880
    image #9
        id: 611
        width: 5120
        height: 2880
    image #10
        id: 672
        width: 5120
        height: 2880
    image #11
        id: 733
        width: 5120
        height: 2880
    image #12
        id: 794
        width: 5120
        height: 2880
    image #13
        id: 855
        width: 5120
        height: 2880
    image #14
        id: 916
        width: 5120
        height: 2880
    image #15
        id: 977
        width: 5120
        height: 2880
```

Thus we can see that in the first image we have additional [XMP metadata](https://www.adobe.com/products/xmp.html). Now I know that I have to modify a little bit my Swift application. It should also add metadata for the first image in the sequence.

```swift
import Foundation
import AppKit
import AVFoundation

extension NSImage {
    @objc var CGImage: CGImage? {
        get {
            guard let imageData = self.tiffRepresentation else { return nil }
            guard let sourceData = CGImageSourceCreateWithData(imageData as CFData, nil) else { return nil }
            return CGImageSourceCreateImageAtIndex(sourceData, 0, nil)
        }
    }
}

let output = "wallpapers-new/output.heic"
let quality = 0.9
var imageData: Data? = nil

if let dir = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask).first {

    let destinationData = NSMutableData()
    if let destination = CGImageDestinationCreateWithData(destinationData, AVFileType.heic as CFString, 16, nil) {
        let options = [kCGImageDestinationLossyCompressionQuality: quality]

        for index in 1...16 {
            let fileURL = dir.appendingPathComponent("wallpapers-new/\(index).png")
            let orginalImage = NSImage(contentsOf: fileURL)

            if let cgImage = orginalImage?.CGImage {

                if index == 1 {
                    let imageMetadata = CGImageMetadataCreateMutable()

                    let imageMetadataTag = CGImageMetadataTagCreate("http://ns.apple.com/namespace/1.0/" as CFString, 
                                                                    "apple_desktop" as CFString, 
                                                                    "solar" as CFString, 
                                                                    CGImageMetadataType.string, 
                                                                    "YnBsaXN0MDDRAQJSc2mvEBADDBAUGBwgJCgsMDQ4PEFF1AQFB" +
                                                                    "gcICQoLUWlRelFhUW8QACNAcO7vOubr3yO/1e+pmkOtXBAB1A" +
                                                                    "QFBgcNDg8LEAEjQFRxqCKOFiAjwCR6waUkDgHUBAUGBxESEws" +
                                                                    "QAiNAVZV4BI4c+CPAEP2uFrMcrdQEBQYHFRYXCxADI0BWtALK" +
                                                                    "mrjwIz/2ObLnx6l21AQFBgcZGhsLEAQjQFfTrJlEjnwjQByrL" +
                                                                    "le1Q0rUBAUGBx0eHwsQBSNAWPrrmI0ISCNAKiwhpSRpc9QEBQ" +
                                                                    "YHISIjCxAGI0BgJff9KDpyI0BENTOsilht1AQFBgclJicLEAc" +
                                                                    "jQGbHdYIVQKojQEq3fAg86lXUBAUGBykqKwsQCCNAbTGmpC2Y" +
                                                                    "RiNAQ2WFOZGjntQEBQYHLS4vCxAJI0BwXfII2B+SI0AmLcjfu" +
                                                                    "C7g1AQFBgcxMjMLEAojQHCnF6YrsxcjQBS9AVBLTq3UBAUGBz" +
                                                                    "U2NwsQCyNAcTcSnimmjCPAGP5E0ASXJtQEBQYHOTo7CxAMI0B" +
                                                                    "xgSADjxK2I8AoalieOTyE1AQFBgc9Pj9AEA0jQHNWsnnMcWIj" +
                                                                    "wEO+oq1pXr8QANQEBQYHQkNEQBAOI0ABZpkFpAcAI8BKYGg/V" +
                                                                    "vMf1AQFBgdGR0hAEA8jQErBKblRzPgjwEMGElBIUO0ACAALAA" +
                                                                    "4AIQAqACwALgAwADIANAA9AEYASABRAFMAXABlAG4AcAB5AII" +
                                                                    "AiwCNAJYAnwCoAKoAswC8AMUAxwDQANkA4gDkAO0A9gD/AQEB" +
                                                                    "CgETARwBHgEnATABOQE7AUQBTQFWAVgBYQFqAXMBdQF+AYcBk" +
                                                                    "AGSAZsBpAGtAa8BuAHBAcMBzAHOAdcB4AHpAesB9AAAAAAAAA" +
                                                                    "IBAAAAAAAAAEkAAAAAAAAAAAAAAAAAAAH9" as CFTypeRef)

                    let success = CGImageMetadataSetTagWithPath(imageMetadata, nil, "xmp:solar" as CFString, imageMetadataTag!)
                    if !success {
                        print("Error!!!")
                    }

                    CGImageDestinationAddImageAndMetadata(destination, cgImage, imageMetadata, options as CFDictionary)

                } else {
                    CGImageDestinationAddImage(destination, cgImage, options as CFDictionary)
                }
            }
        }

        CGImageDestinationFinalize(destination)
        imageData = destinationData as Data

        let outputURL = dir.appendingPathComponent(output)
        try! imageData?.write(to: outputURL)
    }
}
```

Now we can set up our new file as a wallpaper. And this is the result:

<iframe src="https://www.youtube.com/embed/_r0Qxblyz8U?feature=oembed" width="640" height="480" frameborder="0" scrolling="no"></iframe>

**It works!** We can prepare our custom dynamic wallpaper! And it’s pretty easy.

---

Now the question is: what is in the `apple_desktop:solar` attribute? It seems that this is something encoded in base64. We can save value of that attribute as a text file (e.g. `text.base64`) and run following command:

```sh
$ base64 -D text.base64 -o decoded.txt
```

After opening `decoded.txt` file we can notice that it starts from: `bplist00`. So it seems that it’s binary `plist` file. We can run next command:

```sh
$ plutil -convert xml1 decoded.txt
```

And now in `decoded.txt` file we have XML with data which was prepared by Apple.

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
  <dict>
   <key>a</key>
   <real>-10.239758644725045</real>
   <key>i</key>
   <integer>1</integer>
   <key>o</key>
   <integer>1</integer>
   <key>z</key>
   <real>81.775887144809985</real>
  </dict>
  <dict>
   <key>a</key>
   <real>-4.2477344080754564</real>
   <key>i</key>
   <integer>2</integer>
   <key>o</key>
   <integer>1</integer>
   <key>z</key>
   <real>86.33545030477751</real>
  </dict>
  <dict>
   <key>a</key>
   <real>1.3890866331008431</real>
   <key>i</key>
   <integer>3</integer>
   <key>o</key>
   <integer>1</integer>
   <key>z</key>
   <real>90.812670374961954</real>
  </dict>
  <dict>
   <key>a</key>
   <real>7.167168970526129</real>
   <key>i</key>
   <integer>4</integer>
   <key>o</key>
   <integer>1</integer>
   <key>z</key>
   <real>95.307409588765893</real>
  </dict>
  <dict>
   <key>a</key>
   <real>13.08619419164163</real>
   <key>i</key>
   <integer>5</integer>
   <key>o</key>
   <integer>1</integer>
   <key>z</key>
   <real>99.920629632689383</real>
  </dict>
  <dict>
   <key>a</key>
   <real>40.415639464904281</real>
   <key>i</key>
   <integer>6</integer>
   <key>o</key>
   <integer>1</integer>
   <key>z</key>
   <real>129.18652208191958</real>
  </dict>
  <dict>
   <key>a</key>
   <real>53.433472661727741</real>
   <key>i</key>
   <integer>7</integer>
   <key>o</key>
   <integer>1</integer>
   <key>z</key>
   <real>182.23309425497911</real>
  </dict>
  <dict>
   <key>a</key>
   <real>38.793128200638634</real>
   <key>i</key>
   <integer>8</integer>
   <key>o</key>
   <integer>1</integer>
   <key>z</key>
   <real>233.55159195809591</real>
  </dict>
  <dict>
   <key>a</key>
   <real>11.089423171265878</real>
   <key>i</key>
   <integer>9</integer>
   <key>o</key>
   <integer>1</integer>
   <key>z</key>
   <real>261.87159046576664</real>
  </dict>
  <dict>
   <key>a</key>
   <real>5.1845753236736245</real>
   <key>i</key>
   <integer>10</integer>
   <key>o</key>
   <integer>1</integer>
   <key>z</key>
   <real>266.44327370710511</real>
  </dict>
  <dict>
   <key>a</key>
   <real>-6.2483093741227886</real>
   <key>i</key>
   <integer>11</integer>
   <key>o</key>
   <integer>1</integer>
   <key>z</key>
   <real>275.44204536695247</real>
  </dict>
  <dict>
   <key>a</key>
   <real>-12.20770735214888</real>
   <key>i</key>
   <integer>12</integer>
   <key>o</key>
   <integer>1</integer>
   <key>z</key>
   <real>280.07031589401174</real>
  </dict>
  <dict>
   <key>a</key>
   <real>-39.48933951993012</real>
   <key>i</key>
   <integer>13</integer>
   <key>o</key>
   <integer>0</integer>
   <key>z</key>
   <real>309.41857318745144</real>
  </dict>
  <dict>
   <key>a</key>
   <real>-52.753181378799347</real>
   <key>i</key>
   <integer>14</integer>
   <key>o</key>
   <integer>0</integer>
   <key>z</key>
   <real>2.1750965538675473</real>
  </dict>
  <dict>
   <key>a</key>
   <real>-38.04743388682423</real>
   <key>i</key>
   <integer>15</integer>
   <key>o</key>
   <integer>0</integer>
   <key>z</key>
   <real>53.509085812513092</real>
  </dict>
 </array>
</dict>
</plist>
```

We have 16 `dict` elements (one for each image) with four keys:

- a — (probably) some time marker
- i — image index
- o — ? (in current Mojave wallpaper it is always equal 0)
- z — some time marker

Unfortunately I don’t know yet how to convert above float numbers to time markers. If you have some ideas please let me know. Then I will have all mandatory pieces to build macOS application which based on chosen files will generate dynamic wallpaper.

For now I can build only wallpaper which will behave exactly the same way like Apple’s wallpaper.

---

Update: I’ve wrote second part of this article: [**macOS Mojave dynamic wallpapers (II)**](https://writefreely.social/mczachurski/macos-mojave-dynamic-wallpapers-ii).
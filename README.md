# Edit HDR gamma
*Automator App for editing HDR gamma of images in MacOS photo gallery.*

If you take photos on your iPhone 13 and Sony Alpha, you may have noticed that photos taken with the iPhone 13 tend to look brighter and crisper than photos taken with Sony when displayed in the stock Photos app. It harnesses the power of XDR display that is capable to show up to 1600 nits of light output. But photos taken with Sony are displayed in the usual SDR range and look flat.

This application will make your Sony photos magically look best with just one click (ok, that wasn't true, three clicks)

## How to use

0. Install ExifTool by calling `brew install exiftool`
1. Download the application from **Releases** section
2. Put it in **Applications** folder
3. Open your MacOS Photos app
4. Select one or multiple photos taken on your loved DSLR
5. Right click and select **Edit in** in the context menu
6. Select **Other...** and in the files selection dialog box select **Edit HDR gamma** app from Applications folder where you copied it at the step 2
7. **Enter HDR gamma value** dialog will appear with the default value (1.5) or the previous edited value filled in
8. Enter the value 0 to remove the effect or 1.5 for usually good enough result
9. Click **Ok** and enjoy bright, cool images

The next time, **Edit HDR gamma** app will be accessible right from the context menu as the last used application.

## Explanation

Since Apple doesn't provide an official explanation of how photos are taken and stored, I think this is a marketing ploy for amateurs to prefer shooting with iPhones rather than professional cameras. At first I thought that perhaps the iPhone 13 takes photos in the HDR range and saves them in a proprietary format (HEIC) and the Photos app only knowns how to display them, but later I realized that photos saved in JPEG format (8-bit) still look bright and photos saved in HEIC format also have 8-bit which is not enough to store HDR. 

Later I also discovered that both HEIC and JPEG images in subsections contain what is called an HDR gain map, a hidden image that specifies how much "extra" light to display on the top of the normal SDR. When I found a way to split and merge the HDR gain map back I found that changes to the HDR gain map don't affect how the image is displayed at all! The only reason why iPhone's images look "bright" is hidden in the EXIF metadata and HDR gain map is not used to display (the photo editing software still could use it though)

Then I started removing EXIF tags one by one to see if there was one particular EXIF tag affecting the image, and you know, I found the one. ExifTool I used for my investigation doesn't show this tag by default because it is unknown and hidden in the proprietary Apple's "makernotes" section. The command which shows this section is `exiftool -apple:all -u image.jpeg`, and the tag is displayed under the name `Apple 0x0021`. It contains a single floating point value, where 0 means to display the original SDR photo and values greater than 0 extend the standard dynamic range (SDR, 8-bit, 0-255) to the maximum of what could be displayed on XDR Retina displays. I named this tag "HDR Gamma" but that name probably doesn't reflect the truth.

## How it works

I made the Automator app which accepts the file name(s) as input and uses ExifTool to operate only on the "HDRGamma" tag so you may not worry about the quality or metadata loss.

As it was mentioned, the tag is hidden in ExifTool because it is not default, so I use additional config to make it visible and editable:

```
%Image::ExifTool::UserDefined = (
    'Image::ExifTool::Apple::Main' => {
        0x0021 => {
            Name => 'HDRGamma',
            Writable => 'rational64s',
        },
    },
);
```

I create the Apple's "makernote" section using the binary placeholder, containing this only tag, because if this section is missing, the tag is also not editable.

```
printf "\x41\x70\x70\x6c\x65\x20\x69\x4f\x53\x00\x00\x01\x4d\x4d\x00\x01\x00\x21\x00\x0a\x00\x00\x00\x01\x00\x00\x00\x1c\x00\x00\x00\x03\x00\x00\x00\x01" > /tmp/makernotes
```

Then I use ExifTool to operate the image:

```
exiftool -config /tmp/exiftool.config -overwrite_original -if 'not $apple:all' "-makernotes<=/tmp/makernotes" <image>
exiftool -config /tmp/exiftool.config -overwrite_original "-apple:HDRGamma=<gamma>" <image>
```

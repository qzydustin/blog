+++
title = "Optimizing Sony Digital Paper for Chinese"
date = 2024-06-18T22:49:50+00:00

[taxonomies]
categories = ["Device"]
tags = ["devices", "tooling", "fonts"]
+++

The Sony DPT-RP1 is a large-screen E Ink reader built for A4 PDFs. Sony ships the same firmware image in China, Japan, and the United States. A region identifier stored outside the firmware selects the UI language, the input methods, and the default font families, so flashing firmware cannot change the region.

The Japanese build has Japanese and English fonts. The American build has English only, while the Chinese build has Chinese and English. My Japanese unit rendered Chinese characters in a book's table of contents with Japanese glyph variants, or did not render them at all. The Chinese and American builds did not have that problem. Their default font families differ.

<!--more-->

## Why Chinese fails to render

The DPT-RP1 runs Android. Like every Android device it resolves glyphs through a system font configuration file rather than baking fonts into each application. On this device the file is `/system/etc/fonts_split_1_common.xml`. It maps family names (`default`, `sans-serif`, `sst`, `sstjppro`, `dyna`) to font files and defines how the renderer picks a glyph: a full BCP-47 language tag match first, then the language alone, then the first font in file order that contains the glyph.

The Japanese build assigns `SSTJpPro`, a Japanese-oriented CJK font, to `default` and `sans-serif`. Unicode uses one codepoint for many characters shared by Chinese, Japanese, and Korean. The font chooses the regional glyph, so the Japanese font draws the Japanese form. For codepoints it does not contain, Android tries the fallback list. If none of those fonts contains the glyph, the reader shows a blank or a box. The table of contents uses the system default family, which is why its Chinese text is Japanese-looking or missing.

## The fix

The font file lives under `/system`, which is mounted read-only, so this change requires root access. The open-source tool [dpt-tools](https://github.com/HappyZ/dpt-tools) grants root access. Its [setup guide](https://github.com/HappyZ/dpt-tools/wiki) covers installation.

How far you go depends on what you want working:

- Steps 1 and 2 fix the table-of-contents font.
- Step 3 adds Chinese input for search.
- Steps 1 through 3 make the Japanese device behave like another regional build.
- The remaining steps point the system font at `SST` so third-party apps render Chinese.

### Steps

1. Open the "Test Mode" app and change SKU to C/U.

2. Set the system language to CN/US.
   ```shell
   adb shell am start -a android.settings.LOCALE_SETTINGS
   ```

3. (Optional) Set the input method to a Chinese IME.
   ```shell
   adb shell am start -a android.settings.INPUT_METHOD_SETTINGS
   ```

4. Remount `/system` read-write.
   ```shell
   adb shell
   mount -o rw,remount /system
   ```

5. Edit `/system/etc/fonts_split_1_common.xml`. In `<family name="default">` and `<family name="sans-serif">`, replace the `SSTJpPro` entries with `SST`.

   - Original
     ```xml
             <font weight="400" style="normal">SSTJpPro-Light.otf</font>
             <font weight="500" style="normal">SSTJpPro-Regular.otf</font>
             <font weight="700" style="normal">SSTJpPro-Bold.otf</font>
     ```

   - Modified
     ```xml
             <font weight="400" style="normal">SST-Light.otf</font>
             <font weight="500" style="normal">SST-Regular.otf</font>
             <font weight="700" style="normal">SST-Bold.otf</font>
     ```

   The system ships no editor. Pull the file to a host, edit it there, and push it back:
   ```shell
   # Pull the backup files to the local machine
   adb pull /system/etc/fonts_split_1_common.xml fonts_split_1_common.xml.bak

   # Modify the file by using any tools you have
   # Push font configuration to the device's /sdcard/Download directory
   adb push fonts_split_1_common.xml /sdcard/Download

   # Enter the device shell
   adb shell
   mount -o rw,remount /system

   # Move new font configuration files to the system directory
   mv /sdcard/Download/fonts_split_1_common.xml /system/etc/fonts_split_1_common.xml
   chown root:root /system/etc/fonts_split_1_common.xml
   chmod 644 /system/etc/fonts_split_1_common.xml
   ```

6. Reboot.

## Why the fix covers PDF and EPUB

Not every document embeds its own fonts, so the system font setting still matters. A PDF can embed full fonts or only the glyphs it uses. In that case the viewer renders the embedded font regardless of the system configuration. Without embedded CJK fonts, it uses the same `fonts_split_1_common.xml` fallback chain. The table of contents makes this especially visible because the reader draws that panel with the `default` family instead of page-embedded glyphs.

EPUB is reflowable XHTML with CSS. It relies on the reader's installed font stack unless the book declares an embedded font through `@font-face`. Most EPUBs do not embed CJK fonts, because the files are large, so Chinese text falls back to the system families and inherits the same Japanese-preference problem.

Changing the system font covers every document that does not embed its own fonts.

## Backup of fonts_split_1_common.xml

The unmodified file, for reference:

```xml
<?xml version="1.0" encoding="utf-8"?>
<!--
    NOTE: this is the newer (L) version of the system font configuration,
    supporting richer weight selection. Some apps will expect the older
    version, so please keep system_fonts.xml and fallback_fonts.xml in sync
    with any changes, even though framework will only read this file.

    All fonts without names are added to the default list. Fonts are chosen
    based on a match: full BCP-47 language tag including script, then just
    language, and finally order (the first font containing the glyph).

    Order of appearance is also the tiebreaker for weight matching. This is
    the reason why the 900 weights of Roboto precede the 700 weights - we
    prefer the former when an 800 weight is requested. Since bold spans
    effectively add 300 to the weight, this ensures that 900 is the bold
    paired with the 500 weight, ensuring adequate contrast.
-->
<familyset version="22">
    <!-- first font is default -->
    <family name="default">
        <font weight="400" style="normal">SSTJpPro-Light.otf</font>
        <font weight="500" style="normal">SSTJpPro-Regular.otf</font>
        <font weight="700" style="normal">SSTJpPro-Bold.otf</font>
    </family>

    <family name="sst">
        <font weight="400" style="normal">SST-Light.otf</font>
        <font weight="500" style="normal">SST-Regular.otf</font>
        <font weight="700" style="normal">SST-Bold.otf</font>
    </family>

    <family name="sstjppro">
        <font weight="400" style="normal">SSTJpPro-Light.otf</font>
        <font weight="500" style="normal">SSTJpPro-Regular.otf</font>
        <font weight="700" style="normal">SSTJpPro-Bold.otf</font>
    </family>

    <family name="dyna">
        <font weight="400" style="normal">DFHEI5-SONY.ttf</font>
        <font weight="500" style="normal">DFHEI5-SONY.ttf</font>
        <font weight="700" style="normal">DFHEI5-SONY.ttf</font>
    </family>

    <family name="sans-serif">
        <font weight="400" style="normal">SSTJpPro-Light.otf</font>
        <font weight="500" style="normal">SSTJpPro-Regular.otf</font>
        <font weight="700" style="normal">SSTJpPro-Bold.otf</font>
    </family>
</familyset>
```

---
layout: post
title: "Tools of the trade, AKA tools I couldn't do without: 7-Zip"
date: 2011-10-20
categories: technology tools
tags: tools-of-the-trade compression software open-source
author: "Daniel Streefkerk"
excerpt: "A review of 7-Zip, a powerful open-source file archiver with high compression ratio that handles multiple formats including ZIP, GZIP, TAR, ISO, and many others."
---

> **Note:** This article was originally written in October 2011 and is now over 13 years old. Despite that, I still use the latest version of 7-Zip nearly every day.

Another tool that I install straight away on new PCs is 7-Zip. I'm still amazed to see IT pros using unregistered (or dodgy) versions of WinRAR and WinZIP. 7-ZIP is better, faster, and free (open-source).

It can handle lots of formats, including ISO files, which I sometimes find handy.

7-Zip's main features, as per their website. I've highlighted some of the formats that I regularly use 7-Zip with:

> - High compression ratio in [7z format](https://7-zip.org/7z.html) with **LZMA** and **LZMA2** compression  
> - Supported formats:  
>   - Packing / unpacking: 7z, XZ, BZIP2, **GZIP**, **TAR**, **ZIP** and WIM  
>   - Unpacking only: ARJ, CAB, CHM, CPIO, CramFS, DEB, DMG, FAT, HFS, ISO, LZH, LZMA, MBR, **MSI**, NSIS, NTFS, RAR, RPM, SquashFS, UDF, **VHD**, **WIM**, XAR and Z.
> - **For ZIP and GZIP formats, 7-Zip provides a compression ratio that is 2-10% better than the ratio provided by PKZip and WinZip**
> - Strong AES-256 encryption in 7z and ZIP formats  
> - Self-extracting capability for 7z format  
> - Integration with Windows Shell  

**Where:** [https://7-zip.org/](https://7-zip.org/)  
**Licence:** GNU LGPL - [https://7-zip.org/license.txt](https://7-zip.org/license.txt)

**Update 2024:** 7-Zip continues to outperform commercial competitors in benchmark tests, maintaining better compression ratios while being significantly faster, especially when using multi-threading with the LZMA2 algorithm. Recent comparisons confirm that 7-Zip remains the best option for maximum compression efficiency.

> **Security Advisory:** In 2024, a critical vulnerability (CVE-2024-11477) was discovered in 7-Zip that could allow remote code execution. Users should ensure they're running version 24.07 or later, which patches this security issue. Since 7-Zip lacks an automatic update mechanism, manual updates are necessary to stay protected. Always download 7-Zip only from the official website (https://7-zip.org/) to avoid malware.

**Outdated Information:**

- 7-Zip now officially supports Linux natively since version 21.01 (alpha)
- The software has added support for additional archive formats since 2011
- Modern versions offer improved multi-threading performance on newer processors
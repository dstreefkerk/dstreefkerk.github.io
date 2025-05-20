---
layout: post
title: "Batch convert images to B&W with PowerShell and ImageMagick"
date: 2012-08-02
categories: powershell technology
tags: powershell scripting windows imagemagick image-processing
author: "Daniel Streefkerk"
excerpt: "A practical PowerShell script for batch-converting images to black and white using ImageMagick, while properly handling embedded thumbnails."
---

> **Note:** This article was originally written in August 2012 and is now over 12 years old. Due to changes in PowerShell and ImageMagick over time, the described solution may need modifications to work with current versions. Consider this guide as a conceptual reference rather than a current implementation guide.

I knocked together this PowerShell script today to batch-convert 600+ staff photo images to B&W.

*[Image removed during migration: Screenshot showing PowerShell script execution converting images to black and white]*

You'll require the following items installed **and in your path** before this script will work:

1. [ImageMagick](https://imagemagick.org/script/download.php). I used the normal version, not one of the alternate ones.
2. [jhead: EXIF JPEG header manipulation tool](https://www.sentex.ca/~mwandel/jhead/). This requires ImageMagick in order to work.

ImageMagick does the actual conversion to B&W. I played with converting to pure grayscale, but it didn't look good. [This method](https://imagemagick.org/Usage/color_mods/#modulate) instead strips out all colour information by setting saturation to zero:

```
convert.exe {sourcefile} -modulate 100,0 {destinationfile}
```

An issue I came across then is that ImageMagick [doesn't update the embedded JPG thumbnail](https://imagemagick.org/discourse-server/viewtopic.php?f=23&t=20139#p79820). This issue almost stumped me, but then I came across this great little tool called [jhead](https://www.sentex.ca/~mwandel/jhead/). Amongst other things, jhead can regenerate the JPG thumbnail (only if one existed originally):

```
jhead.exe -rgt {filename}
```

Tying it all together is PowerShell:

```powershell
<#
    .SYNOPSIS
    Generate B&W versions of images 
   
       Daniel Streefkerk
    
    THIS CODE IS MADE AVAILABLE AS IS, WITHOUT WARRANTY OF ANY KIND. THE ENTIRE 
    RISK OF THE USE OR THE RESULTS FROM THE USE OF THIS CODE REMAINS WITH THE USER.
    
    Version 10, 02/08/2012
    
    .DESCRIPTION
    
    This script runs though a folder full of images and creates B&W versions of said images
    
    IMPORTANT NOTE: This script requires ImageMagick and jhead. For more information, see my blog
    
    .PARAMETER RootFolder
    The folder within which to process images

    .PARAMETER Recursive
    Also scan subfolders? This is enabled by default
    
    .EXAMPLE
    Process all images in c:temp
    .Generate-BWImages.ps1 -RootFolder c:temp
    
    #>

Param(
  [Parameter(Position=0,Mandatory=$true,ValueFromPipeline=$false,HelpMessage='The folder that contains the photos')]
  [String]
  #[ValidateScript({Test-Path $_ -PathType 'Container'})] 
  $RootFolder,
  [Parameter(Position=1,Mandatory=$false,ValueFromPipeline=$false,HelpMessage='Recurse through all subfolders?')]
  [bool]
  $Recursive=$true
)

cls

# Change these if necessary
$fileExtensions = "*.jpg"
$fileNameSuffix = "_bw" # the text to be appended to the file name to indicate that it has been modified

$files = $null;
$fileCount = 0

# Check if the root folder is a valid folder. If not, try again.
if ((Test-Path $RootFolder -PathType 'Container') -eq $false) {
    Write-Host "'$RootFolder' doesn't seem to be a valid folder. Please try again" -ForegroundColor Red
    break
}

# Get all image files in the folder
if ($Recursive) {
    $files = gci $RootFolder -Filter $fileExtensions -File -Recurse
} else {
    $files = gci $RootFolder -Filter $fileExtensions -File
}
# If there are no image files found, write out a message and quit
if ($files.Count -lt 1) {
    Write-Host "No image files with extension '$fileExtensions' were found in the folder '$RootFolder'" -ForegroundColor Red
    break
}

# Loop through each of the files and process it
foreach ($image in $files) {
    $newFilename = $image.DirectoryName + "\" + $image.BaseName + $fileNameSuffix + $image.Extension
    $imageFullname = $image.FullName

    write-host "Processing image: $imageFullname" -ForegroundColor Green
    & convert.exe $image.FullName -modulate "100,0" $newFilename
    write-host "Updating embedded thumbnail for: $newFilename" -ForegroundColor Green
    & jhead.exe -rgt $newFilename

    $fileCount++
}

Write-Host "$fileCount images processed" -ForegroundColor Yellow
```

## Modern Alternatives and Updates (2025)

Since this article was written in 2012, several aspects have changed:

1. **ImageMagick Command Syntax**: Modern ImageMagick (version 7+) uses a different command structure. The `convert` command is now often called with `magick convert` or simply `magick`.

2. **PowerShell Improvements**: PowerShell 7+ includes better parameter handling and improved pipeline operations that can make scripts like this more efficient.

3. **Alternative Tools**: ExifTool has become a popular alternative to jhead for handling EXIF data and can perform many of the same operations.

## Security Considerations

When using this script or any similar ones:

1. **Input Validation**: Always validate and sanitize folder paths and filenames, especially if they come from user input.

2. **ImageMagick Security**: ImageMagick has had security vulnerabilities in the past. Modern installations should include a security policy file that limits potentially dangerous operations. Always keep ImageMagick updated to the latest version.

3. **Execution Policy**: Be aware of PowerShell's execution policy, which may need to be adjusted to run scripts like this. Use the least permissive policy required.

4. **Consider Using Call Operator**: Modern PowerShell best practices suggest using the call operator (`&`) with proper quoting when invoking external commands, especially when paths contain spaces.
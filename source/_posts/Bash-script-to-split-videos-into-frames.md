---
title: Bash script to split videos into frames
date: 2019-07-10 05:09:27
tags:
---
*by Connor Strang*

I just spent a few hours relearning how to write bash scripts to make a simple utility to split training footage into frames prior to labeling.

For those that just want the code, here it is:

```sh
#!/bin/bash
```

## Backstory

I needed a way to split frames from our training footage into separate frames that we can label to train our neural network. I first considered stepping through the video and screen shotting each frame ... for all of 3 milliseconds. : P

I of course turned to Google and found this [article](https://www.raymond.cc/blog/extract-video-frames-to-images-using-vlc-media-player/) which listed several utilities: 

* [Free Video to JPG Converter](https://www.dvdvideosoft.com/products/dvd/Free-Video-to-JPG-Converter.htm) (Windows-only, Tries to install adware)
* [VLC Media Player](https://www.videolan.org/vlc/index.html)
* [`vlc`](https://wiki.videolan.org/VLC_command-line_help)
* [VirtualDub](http://www.virtualdub.org) (Windows-only)
* [`ffmpeg`](https://ffmpeg.org)

I use a Mac OS computer which leaves me with few free choices other than VLC. I tried VLC's `Scene Video Filter` but like some of VLC's other seemingly basic features (cough cough, stepping backwards a frame), it seemed overly challenging. Disabling or enabling the filter wouldn't take effect without a program restart, and pausing the playback didn't stop the filter from outputting images. Best way to get a clean output from the GUI:

* Make sure filter is disabled.
* Load file and seek to start position.
* Open preferences. Set filter parameters, including rate and output location. Save preferences.
* Close and reopen VLC.
* Open the file again, making sure to select "Continue" in the dialog that hopefully pops up. Do this quickly as VLC is already starting to dump images in the output location.
* Let video play until the stop position.
* Quit VLC.
* Reopen VLC and disable filter.

Jeez! That's a lot of steps for an imperfect result. There will most likely be duplicate or unintended frames from when the file loaded with the filter enabled.

I considered the `vlc` command, however it can still only output frames as fast as the video would normally play.

That brought me to ffmpeg, an incredibly powerful and flexible transcoder (among many things). The one disadvantage is that I didn't want to have to remember all the flags and what order they go in -- because that is important to ffmpeg. With so many features, messing up the positions of the flags that specify input and output parameters can seriously annoy ffmpeg, and be assured, it has an artistic, varied color palette for its pile of error messages!

The easiest solution is to create a bash script that prompts the user for the various parameters, does some basic input verification, and then builds the `ffmpeg` command, which is exactly what I did.

The code is at the top. Feel free to use it, modify it, share it, take credit for it : P (although this might put any chance of a programming career at risk), whatever.

I'm going to put a disclaimer down here, just for kicks:

THE SOFTWARE IS PROVIDED “AS IS”, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

And if you made it this far,
a big ROV for your trouble:

![rov...](https://www.diabgroup.com/~/media/Images/Global%20Images/Landscape/Subsea/DIAB%20Divinycell%20HCP%20syntactic%20foam%20ROV%20673x294.jpg?h=294&la=en-GB&w=673)





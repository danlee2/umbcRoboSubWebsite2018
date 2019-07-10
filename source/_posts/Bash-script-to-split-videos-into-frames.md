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

printf "\n"

OUT_TEMPLATE='frame%04d.png'

# https://stackoverflow.com/questions/5947742/how-to-change-the-output-color-of-echo-in-linux
CLR_PRMT='\033[0;32m'
CLR_ERR='\033[0;31m'
CLR_INFO='\033[0;37m'
CLR_NULL='\033[0m'

ERR_TSFMT="Error: Invalid time format."
ERR_OPTNA="Error: Option is not available."
ERR_FILENF="Error: File does not exist."
ERR_DIRNF="Error: Folder does not exist."
ERR_CMD="Error: A command needed to run this program is missing!"

print_err() {
	printf "$CLR_ERR$1$CLR_NULL\n"
}

print_info() {
	printf "$CLR_INFO$1$CLR_NULL\n"
}

prompt() {
	printf "$CLR_PRMT$1 >> $CLR_NULL"
}

exit() { # override to add a new line XD
	printf "\n"
	if [[ $1 == 0 ]]; then
		command exit 0
	else
		printf "Exiting...\n\n"
		command exit $1
	fi
}

command -v ffmpeg > /dev/null
if [ $? == 1 ]; then
	print_err "$ERR_CMD"
	print_info "Please install 'ffmpeg'."
	exit 1
fi

command -v ffprobe > /dev/null
if [ $? == 1 ]; then
	print_err "$ERR_CMD"
	print_info "Please install 'ffprobe'."
	exit 1
fi

while [[ $SRC_PATH == "" ]]; do
	prompt "Enter path of video"; read SRC_PATH
	if ! [[ -rs $SRC_PATH && ! -d $SRC_PATH ]]; then # file is: readable, size greater than 0
		print_err "$ERR_FILENF"
		SRC_PATH=""
	fi
done

print_info "Found '$(basename $SRC_PATH)'"

FFMPEG_DUMP=$(ffmpeg -i "$SRC_PATH" -hide_banner 2>&1 | sed '$d') # get info about file and suppress "At least one output file..."

# https://trac.ffmpeg.org/wiki/FFprobeTips
FRAMERATE=$( echo $FFMPEG_DUMP | sed "s/.*, \(.*\) tbr.*/\1/" )
DURATION=$( ffprobe -v error -show_entries format=duration -sexagesimal -of default=nw=1:nk=1 $SRC_PATH )
RESOLUTION=$( ffprobe -v error -select_streams v:0 -show_entries stream=width,height -of csv=s=x:p=0 $SRC_PATH )

prompt "Show complete video info? (y)es/(N)o [3s]"; read -t 3
if [[ $REPLY == 'y' || $REPLY == 'yes' ]]; then
	# print complete info
	print_info "$FFMPEG_DUMP"
else
	if ! [[ $REPLY == *'n'* ]]; then printf '\n'; fi # if no reply, then manually move to next line
	# print summary
	print_info "$DURATION of $RESOLUTION at $FRAMERATE fps"
fi

while [[ $DST_PATH == "" ]]; do
	prompt "Enter folder path for output frames"; read DST_PATH
	if ! [[ -d $DST_PATH ]]; then
		print_err "$ERR_DIRNF"
		DST_PATH=""
	fi
done

DST_PATH=$(echo -n "$DST_PATH" | sed 's/\/$//') # strip trailing slash
print_info "Found '$(basename $DST_PATH)/'"

print_info "\nTime format: [HH:]MM:SS[.m] where .m is decimal seconds and [] indicates an optional term"

while [[ $TS_START == "" ]]; do
	prompt "Choose timestamp for start (ENTER for 0:00)"; read TS_START
	if [[ $TS_START == "" ]]; then
		TS_START="00:00:00"
	elif ! [[ $TS_START == *':'* ]]; then
		print_err "$ERR_TSFMT"
		TS_START=""
	fi
done

prompt "Select (e)ndpoint or (d)uration (ENTER for endpoint)"; read END_MODE
if [[ $END_MODE == 'd' || $END_MODE == 'duration' ]]; then
	END_MODE='DUR'

	while [[ $TS_DUR == "" ]]; do
		prompt "Enter a duration (in time format)"; read TS_DUR
		if ! [[ $TS_DUR == *':'* ]]; then
			print_err "$ERR_TSFMT"
			TS_DUR=""
		fi
	done
elif [[ $END_MODE == "" || $END_MODE == 'e' || $END_MODE == 'endpoint' ]]; then
	END_MODE='POINT'

	while [[ $TS_DONE == "" ]]; do
		prompt "Choose a timestamp to stop at (ENTER for end)"; read TS_DONE
		if [[ $TS_DONE == "" ]]; then
			TS_DONE=$DURATION
		elif ! [[ $TS_DONE == *':'* ]]; then
			print_err "$ERR_TSFMT"
			TS_DONE=""
		fi
	done
fi

while [[ $OUTPUT_RATE == "" ]]; do
	prompt "Choose an output rate in fps (ENTER for framerate)"; read OUTPUT_RATE
	if [[ $OUTPUT_RATE == "" ]]; then
		OUTPUT_RATE=$FRAMERATE
	fi

done

print_info "Building command..."

# https://www.raymond.cc/blog/extract-video-frames-to-images-using-vlc-media-player/
# https://trac.ffmpeg.org/wiki/Seeking

# uses -to for both, however without -copyts, the timestamp resets, so -to behaves like a duration
if [[ $END_MODE == 'POINT' ]]; then
	CMD="ffmpeg -ss $TS_START -r $OUTPUT_RATE -i $SRC_PATH -to $TS_DONE -copyts $DST_PATH/$OUT_TEMPLATE -hide_banner"
elif [[ $END_MODE == 'DUR' ]]; then
	CMD="ffmpeg -ss $TS_START -r $OUTPUT_RATE -i $SRC_PATH -to $TS_DUR $DST_PATH/$OUT_TEMPLATE -hide_banner"
fi

echo "$CMD" # use echo to prevent printf from interpreting the regex in the output path

prompt "Run this command? (ENTER for yes, anything else to cancel)"; read RUN_CMD
if [[ $RUN_CMD == "" ]]; then 
	print_info "Running command..."
	$CMD
	exit $?
fi

exit 2
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






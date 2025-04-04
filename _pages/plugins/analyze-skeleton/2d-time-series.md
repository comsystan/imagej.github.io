---
mediawiki: Analyze_Skeleton_2D_time_series
title: Analyze Skeleton 2D time series
---

## Introduction:

This is a Python script that can be run from Fiji (is just imageJ - batteries incuded), using the builtin Jython interpreter Jython is a java implementation of Python which runs natively on the Java Virtual machine. You can use all java classes and imagej classes from it, which is nice. For more info on Jython scripting in Fiji look at [Jython_Scripting](/scripting/jython)

Thanks to Albert Cardona for tips on how to do this.

## Purpose of this script:

There is a limitation in the [ Analyze Skeleton 2D/3D plugin](/plugins/analyze-skeleton) where it can not handle a 2D time series stack as such. It just treats it as a z series, meaning that the results are not as expected since it treats the data as a single 3D dataset instead of as many 2D time points.

## Solution:

To get around this limitation, we can run the "Analyze Skeleton (2D/3D)" command on each individual frame of the time series, since the plugin "Analyze Skeleton (2D/3D)" works as expected on single frames! We need to output the results for each frame, in a sensible readable manner (save as .xls) and make a tagged results image "movie" by converting all the single frame tagged result images into a stack.

## How to use this script:

1. Install the script by putting it in the Fiji plugins directory, then run menu command Plugins - Scripting - Refresh Jythion Scripts. The script name will then appear in the plugins menu (at the bottom)

2. open a movie dataset in Fiji - check image metadata (Bat Cochlea Volume sample image works) check that it is a 2D movie not a 3D z stack: select the mobie image window, then do menu item Image - Properties You might need to change it from a z stack to a time series by changing the numbers there. You can also set the pixel size here if it is wrong in the first instance, so your results are calibrated in micrometers etc. instead of pixels (i hope thats true)

3. Segment objects from the background, for instance using one of the auto threshold methods, such that you have an 8 bit binary image containing zeros for background and 255 for objects.

4. Skeletonize each frame of the movie using the menu command Process - Bnary - Skeletonize Click yes when it asks you if you want to do all the frames/slices in the movie.

5. Run the script: with the skeletonized movie selected run the script by choosing it from the plugins menu. NOTE!!! You MUST change the line in the script: IJ.saveAs("Measurements", ("/Users/dan/Desktop/Results" + str(i) + ".xls") ) so that the path to where you want the results .xls files to be written!!! On windows it might need to be something like: `IJ.saveAs("Measurements", ("C:\\Some Folder\\Whatever name want" + str(i) + ".xls")` )

6. Save the result tagged image movie: TaggedMovie (as a a tiff for example) and the results summary which is in the Log window. The results for each frame were already saved in the last step!

## Code:

```shell
#!/usr/bin/env python

from ij import IJ, ImagePlus
from java.lang import Runtime, Runnable

# the current image
imp = IJ.getImage()
stack = imp.getStack()

# for all the frames in the movie, run  "Analyze Skeleton (2D/3D)" (maybe later in multiple threads)
for i in range(1, imp.getNFrames() + 1):  # remember a python range (1, 10) is the numbers 1 to 9 !
#getNFrames not getNSlices since input data is a 2D time series stack NOT a 3D stack.
  slice = ImagePlus(str(i), stack.getProcessor(i))
  # Execute plugin exactly on this slice i
  IJ.run(slice, "Analyze Skeleton (2D/3D)", "")
  #Save the results for this frame as a .xls file
  IJ.saveAs("Measurements", ("/Users/dan/Desktop/Results" + str(i) + ".xls") )


# turn all the output tagged images into a stack - seems to be in the right order even though all the output frames have the same name
IJ.run("Images to Stack", "name=TaggedMovie title=Tagged use")

IJ.log("Done!")
```

---
title: How to write your own detection algorithm for TrackMate
nav-links:
- title: Edge Feature Analyzers
  url: /plugins/trackmate/extending/custom-edge-feature-analyzer-algorithms
- title: Track Feature Analyzers
  url: /plugins/trackmate/extending/custom-track-feature-analyzer-algorithms
- title: Spot Feature Analyzers
  url: /plugins/trackmate/extending/custom-spot-feature-analyzer-algorithms
- title: Viewers
  url: /plugins/trackmate/extending/custom-viewers
- title: Actions
  url: /plugins/trackmate/extending/custom-actions
- title: Detection Algorithms
  url: /plugins/trackmate/extending/custom-detection-algorithms
- title: Segmentation Algorithms
  url: /plugins/trackmate/extending/custom-segmentation-algorithms
- title: Particle-Linking Algorithms
  url: /plugins/trackmate/extending/custom-particle-linking-algorithms
---

## Introduction

Welcome to the most useful and also unfortunately the hardest part in this tutorial series on how to extend [TrackMate](/plugins/trackmate) with custom modules.

The detection algorithms in TrackMate are basic: they are all based or approximated from the {% include wikipedia title='Blob detection#The_Laplacian_of_Gaussian' text='Laplacian of Gaussian'%} technique. They work well even in the presence of noise for round or spherical and well separated objects. As soon as you move away from these requirements, you will feel the need to implement your own custom detector.

This is the subject of this tutorial, which I promised to be rather difficult. Not because implementing a custom detection algorithm is difficult. It *is* difficult, even very difficult if you are not familiar with the [ImgLib2](/libs/imglib2) library. But we will skip this difficulty here by not making a true detector, but just a dummy one that returns detections irrespective of the image content. This involved task is left to your Java and ImgLib2 skills.

No, this tutorial will be difficult because contrary to the previous ones, we need to do a lot of work even for just a dummy detector. The reason for this comes from our desire to have a nice and tidy integration in TrackMate. The custom detector we will write will be a first-class citizen of TrackMate, and this means several things: Not only it must be able to provide a proper detection, but it must also

-   offer the user some configuration options, in a nice GUI;
-   check that the user entered meaningful detection parameters;
-   enable the saving and loading of these parameters to XML.

We did not have to care when implementing a [custom action](/plugins/trackmate/custom-actions), but now we do.

Let's get started with the easiest part, the detection algorithm.

## The {% include github org='fiji' repo='TrackMate' branch='master' source='fiji/plugin/trackmate/detection/SpotDetector.java' label='SpotDetector' %} interface

### A detector instance operates on a single frame

The detection part itself is implemented in a class that implements the {% include github org='fiji' repo='TrackMate' branch='master' source='fiji/plugin/trackmate/detection/SpotDetector.java' label='SpotDetector' %} interface. Browsing there, you will see that it is just a specialization of an output algorithm from [ImgLib2](/libs/imglib2). We are required to spit out a `List<Spot>` that represents the list of detection (one `Spot` per detection) for a **single frame**.

This is important: <u>an instance of your detector is supposed to work on a single frame</u>. TrackMate will generate as many instances of the detector per frame it has to operate on. This facilitates development, but also multithreading: TrackMate fires one detector per thread it has access to, and this is done without you having to worry about it. TrackMate will bundle the outputs of all detectors in a thread-safe manner.

It is the work of the detector factory to provide each instance with the data required to segment a specific frame. But we will see how this is done below.

### A SpotDetector *can be* {% include github org='imglib' repo='imglib2-algorithm' branch='master' source='net/imglib2/algorithm/MultiThreaded.java' label='multithreaded' %}

So TrackMate offers you a turnkey multithreaded solution: If you have a computer with 12 cores and 50 frames to segment, TrackMate will fire 12 SpotDetectors at once and process them concurrently.

But let's say that you have 24 cores and only 6 frames to segment. You can exploit this situation by letting your concrete instance of SpotDetector implement the ImgLib2 {% include github org='imglib' repo='imglib2-algorithm' branch='master' source='net/imglib2/algorithm/MultiThreaded.java' label='MultiThreaded' %}. interface. In that case, TrackMate will still fire 6 SpotDetector instances (one for each frame), but will allocate 4 threads to each instance, and get an extra kick in speed.

Of course, you have to devise a clever multithreading strategy to operate concurrently on a single frame. For instance, you could divide the image into several blocks and process them in parallel. Or delegate to sub-algorithms that are multithreaded; check for instance the {% include github org='fiji' repo='TrackMate' branch='master' source='fiji/plugin/trackmate/detection/LogDetector.java' label='LogDetector' %} code.

### Detection results are represented by {% include github org='fiji' repo='TrackMate' branch='master' source='fiji/plugin/trackmate/Spot.java' label='Spots' %}

{% include github org='fiji' repo='TrackMate' branch='master' source='fiji/plugin/trackmate/Spot.java' label='Spots' %} are used to represent detection results: one detection = one spot. By convention, a detection algorithm must provide *at least* the following numerical feature to each spot:

-   The X, Y, Z coordinates, obviously. What is not that obvious is that TrackMate uses only image coordinates. This means that if your image has a physical calibration in µm (*e.g.* 0.2 µm/pixels in X,Y), the spot coordinates must be in µm[^1]. If you have just a 2D image, use 0 for the Z position, but it must not be omitted.
-   A quality value, that reflects the quality of the detection itself. It must be a real, positive number, that reflects how confident your detection algorithm is that the found detection is not spurious. The larger the more confident.
-   The spot radius, representing in physical units, the approximate size of the image structure that was detected. TrackMate default detectors do not have an automatic size detection feature, so they ask the user what is the most likely size of the structures they should detect, tune themselves to this size, and set all the radius of the detections to be the one entered by the user.

Any omission will trigger errors at runtime.

A note before we move on: Starting with version 7, TrackMate introduced many new features, including a supplemental spot factory hierarchy that allows tuning how the time-points are processed. Either one by one each by a separate instance of a `SpotDetector`, like it is the case here, or all at once. This is explained in the [next section](/plugins/trackmate/custom-segmentation-algorithms) of this developer tutorial on implementing segmentation algorithm and using the new v7 API.

### A dummy detector that returns spiraling spots

For this tutorial we will build a dummy detector, that actually fully ignores the image content and just create spots that seem to spiral out from the center of the image. A real detector would require you to hone your [ImgLib2](/libs/imglib2) skills; check the {% include github org='fiji' repo='TrackMate' branch='master' source='fiji/plugin/trackmate/detection/LogDetector.java' label='LogDetector' %} code for an example.

Below is the source code for the dummy detector. You can also find it {% include github org='fiji' repo='TrackMate-examples' branch='master' source='plugin/trackmate/examples/detector/SpiralDummyDetector.java' label='online' %}. Let's comment a bit on this:

#### The type parameter `< T extends RealType< T > & NativeType< T >>`

Instances of SpotDetector are parametrized with a generic type `T` that must extends {% include github repo='imglib' branch='master' path='src/main/java/net/imglib2/type/numeric/RealType.java' label='RealType' %} and {% include github repo='imglib' branch='master' path='src/main/java/net/imglib2/type/NativeType.java' label='NativeType' %}. These are the bounds for all the scalar types based on native types, such us `float`, `int`, `byte`, etc...

This is the type of the image data we are to operate on.

#### The constructor

Since the {% include github org='fiji' repo='TrackMate' branch='master' source='fiji/plugin/trackmate/detection/SpotDetector.java' label='SpotDetector' %} interface gives little constraint on inputs, all of them must be provided at construction time in the constructor. Keep in mind that we have one instance per frame, so we must know what frame we are to process.

Normal detectors would be fed with a reference to the image data *for this very single frame*. Here we do not care for image content, so it is not there. But we will speak of this more when discussing the factory.

Because TrackMate can also be tuned to operate only on a ROI, the instance receives an {% include github repo='imglib' branch='master' path='src/main/java/net/imglib2/Interval.java' label='Interval' %} that represent the bounding box **in pixel coordinates** of the ROI the user selected. Here, we just use it to center the spirals.

Because we must store the *physical coordinates*' in the spots we create, we need a calibration array to convert pixel coordinates to physical ones. That is the role of the `double[]calibration` array, and it contains the pixel sizes along X, Y and Z.

#### The {% include github repo='imglib' branch='master' path='imglib2/algorithm/Algorithm.java' label='Algorithm' %} methods

`checkInput()` checks that the parameters passed are OK prior to processing, and returns `false` if they are not. `process()` does all the hard work, and return `false` if something goes wrong.

If any of these two methods returns `false`, you are expected to document what went wrong in an error message that can be retrieved through `getErrorMessage()`.

#### The {% include github repo='imglib' branch='master' path='algorithms/core/src/main/java/net/imglib2/algorithm/OutputAlgorithm.java' label='OutputAlgorithm' %} method

This one just asks us to return the results as a list of spots. It must be a field of your instance, that is ideally instantiated and built in the `precess()` method. The `getResult()` method exposes this list.

#### The {% include github repo='imglib' branch='master' path='algorithms/core/src/main/java/net/imglib2/algorithm/Benchmark.java' label='Benchmark' %} method

Well, we just want to know how much time it took. Note that all of these are the usual suspects of an ImgLib2 generic algorithm, so they should not confuse you.

#### The code itself

```java
    package plugin.trackmate.examples.detector;
    
    import java.util.ArrayList;
    import java.util.List;
    
    import net.imglib2.Interval;
    import net.imglib2.type.NativeType;
    import net.imglib2.type.numeric.RealType;
    import fiji.plugin.trackmate.Spot;
    import fiji.plugin.trackmate.detection.SpotDetector;
    
    public class SpiralDummyDetector< T extends RealType< T > & NativeType< T >> implements SpotDetector< T >
    {
    
        private static final double RADIAL_SPEED = 3d; // pixels per frame
    
        // radians per frame
        private static final double ANGULAR_SPEED = Math.PI / 10;
    
        // in image units
        private static final double SPOT_RADIUS = 1d;
    
        /** The width if the ROI. */
        private final long width;
    
        /** The height if the ROI. */
        private final long height;
    
        /** The X coordinates of the ROI. */
        private final long xstart;
    
        /** The Y coordinates of the ROI. */
        private final long ystart;
    
        /** The pixel sizes in the 3 dimensions. */
        private final double[] calibration;
    
        /** The frame we operate in. */
        private final int frame;
    
        /** Holder for the results of detection. */
        private List< Spot > spots;
    
        /** Error message holder. */
        private String errorMessage;
    
        /** Holder for the processing time. */
        private long processingTime;
    
        /*
         * CONSTRUCTOR
         */
    
        public SpiralDummyDetector( final Interval interval, final double[] calibration, final int frame )
        {
            // Take the ROI box from the interval parameter.
            this.width = interval.dimension( 0 );
            this.height = interval.dimension( 1 );
            this.xstart = interval.min( 0 );
            this.ystart = interval.min( 1 );
            // We will need the calibration to convert to physical units.
            this.calibration = calibration;
            // We need to know what frame we are in.
            this.frame = frame;
        }
    
        /*
         * METHODS
         */
    
        @Override
        public List< Spot > getResult()
        {
            return spots;
        }
    
        @Override
        public boolean checkInput()
        {
            // Nothing to test, it's all good.
            return true;
        }
    
        @Override
        public boolean process()
        {
            final long start = System.currentTimeMillis();
            spots = new ArrayList< Spot >();
    
            /*
             * This dummy detector creates spots that spiral out from the center of
             * the specified ROI. It spits a new spiral every 10 frames.
             */
    
            final int x0 = ( int ) ( width / 2 + xstart );
            final int y0 = ( int ) ( height / 2 + ystart );
    
            int t = frame;
            int nspiral = 0;
            while ( t >= 0 )
            {
                final double r = t * RADIAL_SPEED;
                final double phi0 = nspiral * Math.PI / 4;
                final double phi = t * ANGULAR_SPEED + phi0;
    
                // Spot in pixel coordinates.
                final double x = x0 + r * Math.cos( phi );
                final double y = y0 + r * Math.sin( phi );
    
                // But we want to create spots in image coordinates:
                final double xpos = x * calibration[ 0 ];
                final double ypos = y * calibration[ 1 ];
                final double zpos = 0d;
    
                // Create the spot.
                final Spot spot = new Spot( xpos, ypos, zpos, SPOT_RADIUS, 1d / ( nspiral + 1d ) );
                spots.add( spot );
    
                // Loop to another spiral.
                t = t - 10;
                nspiral++;
            }
    
            final long end = System.currentTimeMillis();
            this.processingTime = end - start;
            return true;
        }
    
        @Override
        public String getErrorMessage()
        {
            /*
             * If something wrong happens while you #checkInput() or #process(),
             * state it in the errorMessage field.
             */
            return errorMessage;
        }
    
        @Override
        public long getProcessingTime()
        {
            return processingTime;
        }
    
    }
```

And that's about it.

Now for something completely different, we move to the factory class that instantiates this detector.

## The {% include github org='fiji' repo='TrackMate' branch='master' source='fiji/plugin/trackmate/detection/SpotDetectorFactory.java' label='SpotDetectorFactory' %} interface

The SpotDetectorFactory concrete implementation is the class that needs to be annotated with the [SciJava](/libs/scijava) annotation. For instance:

```java
@Plugin( type = SpotDetectorFactory.class )
public class SpiralDummyDetectorFactory< T extends RealType< T > & NativeType< T >> implements SpotDetectorFactory< T >
```

Note that we have to deal with the same type parameter than for the SpotDetector instance.

I will skip all the TrackMateModule methods we have seen over and over in this tutorial series. There is nothing new here, they all have the same role. The difficult and interestings parts are linked to what we introduced above. Basically we need to provide a logic for passing the raw image data, for saving/loading to XML, for querying the user for parameters, and checking them.

### Getting the raw image data

Since the TrackMateModule concrete implementation must have a blank constructor, there must be another way to pass required arguments to factories. For SpotDetector factories, this role is played by the `setTarget` method:

```java
@Override
public boolean setTarget( final ImgPlus< T > img, final Map< String, Object > settings )
```

The raw image data is returned as an `ImgPlus`, that can be seen as the [ImgLib2](/libs/imglib2) equivalent of ImageJ's {% include github org='imagej' repo='ImageJ' branch='master' source='ij/ImagePlus.java' label='ImagePlus' %}. It contains the pixel data for all available dimensions (all X, Y, Z, C, T if any), plus the spatial calibration we need to operate in physical units. The concrete factory must be able to extract from this ImgPlus the data useful for the SpotDetectors it will instantiate, keeping in mind that each SpotDetector operates on one frame.

The second argument is the settings map for this specific detector. It takes the shape of a map with string keys and object values, that can be cast to whatever relevant class. The concrete factory must be able to check that all the required parameters are there, and have a valid class, and to feed to the SpotDetector instances. We will see below that the user provides them through a configuration panel.

### Getting detection parameters through a configuration panel

For a proper TrackMate integration, we need to provide a means for users to tune the detector they chose. And since TrackMate was built first to be used through a GUI, we need to create a GUI element for this task: a configuration panel. The class that does that in TrackMate is {% include github org='fiji' repo='TrackMate' branch='master' source='fiji/plugin/trackmate/gui/components/ConfigurationPanel.java' label='ConfigurationPanel' %}. It is an abstract class that extends JPanel, and that adds two methods to display a settings map and return it.

Each SpotDetectorFactory has its own configuration panel, which must be instantiated and returned through:

```java
@Override
public ConfigurationPanel getDetectorConfigurationPanel( final Settings settings, final Model model )
```

The GUI panel has access to the model and settings objects, and can therefore display some relevant information.

This is a difficult part because you have to write a GUI element. GUIs are excruciating long and painfully hard to write, at least if you want to get them right. Check the {% include github org='fiji' repo='TrackMate' branch='master' source='fiji/plugin/trackmate/gui/components/detector/LogDetectorConfigurationPanel.java' label='configuration panel of the LOG detector' %} for an example.

### Checking the validity of parameters

There is a layer of methods that allows checking for the parameter validity. Normally you don't need to, since you write the configuration panel for the detector you also develop, but I have found this to be useful to find errors early. Parameter checking is done after user edition, loading and saving.

Three methods are at play:

```java
public Map< String, Object > getDefaultSettings();

public boolean checkSettings( final Map< String, Object > settings );

public String getErrorMessage();
```

The `getDefaultSettings()` method return a new settings map initialized with default values. It must contain all the required parameter keys, and nothing more. The `checkSettings()` method does the actual parameter checking. It must check that all the required parameters are there, that they have the right class, and that there is no spurious mapping in the map. Should any of these defects be found, it returns `false`. Finally, `getErrorMessage()` should return a meaningful error message if the check failed.

### Saving to and loading from XML

TrackMate tries to save as much information as possible when saving to XML. The save file should contain at the very least the tracking results, but it should also include the parameters that help creating these results. The detection algorithm parameters should therefore be included.

You have to provide the means to save and load this parameters, since they are specific to the detector you write. This is done through the two methods:

```java
public boolean marshall( final Map< String, Object > settings, final Element element );
    
public boolean unmarshall( final Element element, final Map< String, Object > settings );
```

#### Marshalling

Marshalling is the action of serializing a java object to XML. TrackMate relies on the [JDom library](http://www.jdom.org/) to do so, and it greatly simplifies the task.

The settings map that the `marshall` method receives is the settings map to save. You can safely assume it has been successfully checked. The element parameter is a [JDom element](http://www.jdom.org/docs/apidocs/org/jdom2/Element.html), and it must contain eveything you want to save from the detector, as attribute or child elements. Here is what you must put in it:

-   You must at the very least set an attribute that has for key {% include github org='fiji' repo='TrackMate' branch='master' source='fiji/plugin/trackmate/detection/DetectorKeys.java\#L35' label='`"DETECTOR_NAME"`' %} and value the SpotDetectorFactory key (the one you get with the `getKey()`) method[^2]. This will be used in turn when loading from XML, to retrieve the right detector you used.
-   If something goes wrong when saving, then the `marshall` method must return `false`, and you must provide a meaningful error message for the `getErrorMessage()` method.
-   Everything else is pretty much up to you. There is a {% include github org='fiji' repo='TrackMate' branch='master' source='fiji/plugin/trackmate/io/IOUtils.java' label='helper method in IOUtils' %} that you can use to serialize single parameters. Check the {% include github org='fiji' repo='TrackMate' branch='master' source='fiji/plugin/trackmate/detection/LogDetectorFactory.java\#L201' label='LogDetectorFactory marshall method' %} for an example.

#### Unmarshalling

Unmarshalling is just the other way around. You get a map that you must first clear, then build from the JDom element specified. You can safely assume that the XML element you get was built from the `marshall` method of the same SpotDetectorFactory. TrackMate makes sure the right `unmarshall` method is called.

There are a few help methods around to help you with reading from XML. For instance, check all the `read*Attribute` of the {% include github org='fiji' repo='TrackMate' branch='master' source='fiji/plugin/trackmate/io/IOUtils.java' label='IOUtils' %} class. It is also a good idea to call the `checkSettings` method with the map you just built.

Check again the {% include github org='fiji' repo='TrackMate' branch='master' source='fiji/plugin/trackmate/detection/LogDetectorFactory.java\#L213' label='LogDetectorFactory unmarshall method' %} for an example.

### Instantiating spot detectors

And finally, the method that gives its name to this factory:

```java
public SpotDetector< T > getDetector( final Interval interval, final int frame )
```

This will be called repeatedly by TrackMate to generate as many SpotDetector instances as there is frames in the raw data to segment. The two parameters specify the ROI the user wants to operate on as an {% include github repo='imglib' branch='master' path='core/src/main/java/net/imglib2/Interval.java' label='Imglib2 interval' %}, and the target frame. So you need to process and bundle:

-   this interval and this frame;
-   the raw image data and settings map received from the `setTarget` method

in the parameters required to instantiate a new SpotDetector.

Because the dummy example we use in this tutorial is not such an enlightenment, I copy here the code from the LogDetectorFactory. It shows how to extract parameters from a settings map, and how to access the relevant data frame in a possibly 5D image:

```java
        @Override
        public SpotDetector< T > getDetector( final Interval interval, final int frame )
        {
            final double radius = ( Double ) settings.get( KEY_RADIUS );
            final double threshold = ( Double ) settings.get( KEY_THRESHOLD );
            final boolean doMedian = ( Boolean ) settings.get( KEY_DO_MEDIAN_FILTERING );
            final boolean doSubpixel = ( Boolean ) settings.get( KEY_DO_SUBPIXEL_LOCALIZATION );
            final double[] calibration = TMUtils.getSpatialCalibration( img );
    
            RandomAccessible< T > imFrame;
            final int cDim = TMUtils.findCAxisIndex( img );
            if ( cDim < 0 )
            {
                imFrame = img;
            }
            else
            {
                // In ImgLib2, dimensions are 0-based.
                final int channel = ( Integer ) settings.get( KEY_TARGET_CHANNEL ) - 1;
                imFrame = Views.hyperSlice( img, cDim, channel );
            }
    
            int timeDim = TMUtils.findTAxisIndex( img );
            if ( timeDim >= 0 )
            {
                if ( cDim >= 0 && timeDim > cDim )
                {
                    timeDim--;
                }
                imFrame = Views.hyperSlice( imFrame, timeDim, frame );
            }
            final LogDetector< T > detector = new LogDetector< T >( imFrame, interval, calibration, radius, threshold, doSubpixel, doMedian );
            detector.setNumThreads( 1 ); // in TrackMate context, we use 1 thread
            // per detector but multiple detectors
            return detector;
        }
```

### The code for the dummy spiral generator factory

And here is the full code for this tutorial example. It is the ultimate simplification of a SpotDetectorFactory, and was careful to strip anything useful by first ignoring the image content, second by not using any parameter. You can also find it {% include github org='fiji' repo='TrackMate-examples' branch='master' source='plugin/trackmate/examples/detector/SpiralDummyDetectorFactory.java' label='online' %}.

```java
    package plugin.trackmate.examples.detector;
    
    import ij.ImageJ;
    import ij.ImagePlus;
    
    import java.util.Collections;
    import java.util.Map;
    
    import javax.swing.ImageIcon;
    
    import net.imglib2.Interval;
    import net.imglib2.meta.ImgPlus;
    import net.imglib2.type.NativeType;
    import net.imglib2.type.numeric.RealType;
    
    import org.jdom2.Element;
    import org.scijava.plugin.Plugin;
    
    import fiji.plugin.trackmate.Model;
    import fiji.plugin.trackmate.Settings;
    import fiji.plugin.trackmate.TrackMatePlugIn_;
    import fiji.plugin.trackmate.detection.SpotDetector;
    import fiji.plugin.trackmate.detection.SpotDetectorFactory;
    import fiji.plugin.trackmate.gui.ConfigurationPanel;
    import fiji.plugin.trackmate.util.TMUtils;
    
    @Plugin( type = SpotDetectorFactory.class )
    public class SpiralDummyDetectorFactory< T extends RealType< T > & NativeType< T >> implements SpotDetectorFactory< T >
    {
    
        static final String INFO_TEXT = "<html>This is a dummy detector that creates spirals made of spots emerging from the center of the ROI. The actual image content is not used.</html>";
    
        private static final String KEY = "DUMMY_DETECTOR_SPIRAL";
    
        static final String NAME = "Dummy detector in spirals";
    
        private double[] calibration;
    
        private String errorMessage;
    
        @Override
        public String getInfoText()
        {
            return INFO_TEXT;
        }
    
        @Override
        public ImageIcon getIcon()
        {
            return null;
        }
    
        @Override
        public String getKey()
        {
            return KEY;
        }
    
        @Override
        public String getName()
        {
            return NAME;
        }
    
        @Override
        public SpotDetector< T > getDetector( final Interval interval, final int frame )
        {
            return new SpiralDummyDetector< T >( interval, calibration, frame );
        }
    
        @Override
        public boolean setTarget( final ImgPlus< T > img, final Map< String, Object > settings )
        {
            /*
             * Well, we do not care for the image at all. We just need to get the
             * physical calibration and there is a helper method for that.
             */
            this.calibration = TMUtils.getSpatialCalibration( img );
            // True means that the settings map is OK.
            return true;
        }
    
        @Override
        public String getErrorMessage()
        {
            /*
             * If something is not right when calling #setTarget (i.e. the settings
             * maps is not right), this is how we get an error message.
             */
            return errorMessage;
        }
    
        @Override
        public boolean marshall( final Map< String, Object > settings, final Element element )
        {
            /*
             * This where you save the settings map to a JDom element. Since we do
             * not have parameters, we have nothing to do.
             */
            return true;
        }
    
        @Override
        public boolean unmarshall( final Element element, final Map< String, Object > settings )
        {
            /*
             * The same goes for loading: there is nothing to load.
             */
            return true;
        }
    
        @Override
        public ConfigurationPanel getDetectorConfigurationPanel( final Settings settings, final Model model )
        {
            // We return a simple configuration panel.
            return new DummyDetectorSpiralConfigurationPanel();
        }
    
        @Override
        public Map< String, Object > getDefaultSettings()
        {
            /*
             * We just have to return a new empty map.
             */
            return Collections.emptyMap();
        }
    
        @Override
        public boolean checkSettings( final Map< String, Object > settings )
        {
            /*
             * Since we have no settings, we just have to test that we received the
             * empty map. Otherwise we generate an error.
             */
            if ( settings.isEmpty() ) { return true; }
            errorMessage = "Expected the settings map to be empty, but it was not: " + settings + '\n';
            return false;
        }
    }
```

## Wrapping up

Ouf! That was a lot of information and a lot of coding for a single piece of functionality. But all of these painful methods make your detector a first-class citizen of TrackMate. "/develop/native-libraries" detectors use the exact same logic.

Here is what our dummy example looks. To maximize your user experience, I let it run on a 512 x 512 x 200 frames image, and tracked them.

_JY_

![](/media/plugins/trackmate/trackmatecustomdetector-01.gif)

## References

[^1]: The reason behind this is that TrackMate wants to break free of the source data. Keeping all the coordinates in physical units allow exchanging results without having to keep a reference to the original image.

[^2]: Careful, this will not be mandatory in TrackMate v2.3.0

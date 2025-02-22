---
title: ImgLib2 Examples
section: Explore:Libraries:ImgLib2
name: ImgLib2 Examples
team-maintainers: ["@StephanPreibisch", "@ctrueden"]
source-url: https://github.com/imglib/imglib2-tutorials
---

This page shows eight increasingly complex examples of how to program with
ImgLib2. The intention of these examples are not to explain ImgLib2 concepts,
but rather to give some practical hints how to work with the library and to
grasp the principles in a learning-by-doing way.

## Jupyter notebook

This tutorial is also available in Jupyter notebook form [here](https://nbviewer.jupyter.org/github/imagej/tutorials/blob/master/notebooks/3-Advanced-Topics/2-ImgLib2-in-Detail.ipynb)!

## Introduction & required files

All examples presented on this page are always entire classes including a main method to run them. Simply copying them into your favorite editor (e.g. the [Script Editor](/scripting/script-editor)) and compile & run them. The required Java libraries (jar files) are part of [ImageJ2](/software/imagej2) and can be found in `ImageJ2.app/jars/`:

-   imglib2 (the core)
-   imglib2-algorithm (algorithms implemented in ImgLib2)
-   imglib2-algorithm-gpl (for example 6b and 6c: GPL-licensed algorithms implemented in ImgLib2—ships with [Fiji](/software/fiji) only, not plain [ImageJ2](/software/imagej2), for [licensing](/licensing) reasons)
-   imglib2-ij (the ImageJ interaction)
-   imglib2-realtransform (for example 8)
-   scifio (for reading and writing files)
-   ij (ImageJ core, used for display)

Alternately, you can access the examples from the {% include github org='imglib' repo='imglib2-tutorials' label='ImgLib-tutorials Git repository' %}. After cloning the source code, open the project in your favorite IDE. See [Developing ImgLib2](/libs/imglib2/developing) for further details.

## Example 1 - Opening, creating and displaying images

The first example illustrates the most basic operations of opening, creating, and displaying image content in ImgLib2. It will first focus on entires images (`Img<T>`), but also show how to display subsets only.

### Example 1a - Wrapping ImageJ images

If you are already an ImageJ programmer, you might find it the easiest way to simply wrap an ImageJ image into ImgLib2. Here, the data is not copied, so editing the image in ImgLib2 will also modify the ImageJ ImagePlus.

Internally, we use a compatibility `Img` to represent the data which is as fast as ImageJ but in the case of higher dimensionality (&gt;2d) is slower than ImgLib2 can do with the `ArrayImg`. Furthermore you are limited in dimensionality (2d-5d), in the type of data (`UnsignedByteType`, `UnsignedShortType`, `FloatType` and `ARGBType`) and maximal size of each 2d-plane (max. 46000x46000).

{% include code org='imglib' repo='imglib2-tutorials' branch='master'
                        path='src/main/java/Example1a.java' line-start='34' %}

### Example 1b - Opening an ImgLib2 image

The typical way to open an image in ImgLib2 is to make use of the [SCIFIO](https://scif.io/) importer. Below you see two examples of how to open an image as *(a)* its own type (e.g. `UnsignedByteType`) and *(b)* as float (`FloatType`). For *(a)* we assume, however, that the file contains some real valued numbers as defined by the interface `RealType`. Color images are opened as well and color is represented as its own dimension (like in the ImageJ Hyperstacks).

Note that for *(a)* we use an `ArrayImg` to hold the data. This means the data is held in one single java basic type array which results in optimal performance. The absolute size of image is, however, limited to 2^31-1 (\~2 billion) pixels. The type of `Img` to use is set by passing an `ImgOptions` configuration when calling the `ImgOpener`.

In *(b)* we use a `CellImg` instead. It partitions the image data into n-dimensional cells each holding only a part of the data. Further, SCIFIO takes care of caching cells in and out of memory as needed, greatly reducing the memory requirement to work with very large images.

The SCIFIO importer also requires `Type`s that implement `NativeType`, which means it is able to map the data into a Java basic type array. All available `Type`s until now are implementing `NativeType`, if you want to work with some self-developed `Type` it would be easiest to copy the opened `Img` afterwards. Please also note that until now, the only `Img` that supports non-native types is the `ListImg` which stores every pixel as an individual object!

**Important**: it does not matter which type of `Img` you use to hold the data as we will use `Iterator`s and `RandomAccess`es to access the image content. It might be, however, important if you work on two `Img` at the same time using `Iterator`s, see Example2.

{% include code org='imglib' repo='imglib2-tutorials' branch='master'
                        path='src/main/java/Example1b.java' line-start='34' %}

### Example 1c - Creating a new ImgLib2 image

Another important way to instantiate a new ImgLib2 `Img` is to create a new one from scratch. This requires you to define its `Type` as well as the `ImgFactory` to use. It does additionally need one instance of the `Type` that it is supposed to hold.

Once you have one instance of an `Img`, it is very easy to create another one using the same `Type` and `ImgFactory`, even if it has a different size. Note that the call `img.firstElement()` returns the first pixel of any `Iterable`, e.g. an `Img`.

{% include code org='imglib' repo='imglib2-tutorials' branch='master'
                        path='src/main/java/Example1c.java' line-start='34' %}

### Example 1d - Displaying images partly using Views

By using the concept of `Views` it is possible to display only parts of the image, display a rotated view, and many more cool things. Note that you can also concatenate them. `Views` are much more powerful than shown in this example, they will be increasingly used throughout the examples.

A `View` almost behaves similar to an `Img`, and in fact they share important concepts. Both are `RandomAccessible`, and `Views` that are not infinite are also an `Interval` (i.e. those `Views` have a defined size) and can therefore be made `Iterable` (see example 2c). In ImgLib2, all algorithms are implemented for abstract concepts like `RandomAccessible`, `Iterable` or `Interval`. This enables us, as can be seen below, to display a `View` the exact same way we would also display an `Img`.

<img src="/media/libs/imglib2/imglib2example1d.jpg" width="780"/> *Shows the original image, the View of an interval, as well as the by 90 degree rotated version of the view. Note that only the original image in kept in memory, both Views are completely virtual.*

{% include code org='imglib' repo='imglib2-tutorials' branch='master'
                        path='src/main/java/Example1d.java' line-start='34' %}

## Example 2 - How to use Cursor, RandomAccess and Type

The following examples illustrate how to access pixels using `Cursor` and `RandomAccess`, their basic properties, and how to modify pixel values using `Type`.

Accessing pixels using a `Cursor` means to iterate all pixels in a way similar to iterating Java collections. However, a `Cursor` only ensures to visit each pixel exactly once, the order of iteration is not fixed in order to optimize the speed of iteration. This implies that that the order of iteration on two different `Img` is not necessarily the same, see [example 2b](#example-2b-duplicating-an-img-using-a-different-imgfactory)! `Cursor`s can be created by any object that implements `IterableInterval`, such as an `Img`. `Views` that are not infinite can be made iterable (see example 2c). *Note that in general a `Cursor` has significantly higher performance than a `RandomAccess` and should therefore be given preference if possible*.

In contrast to iterating image data, a `RandomAccess` can be placed at arbitrary locations. It is possible to set them to a specific n-dimensional coordinate or move them relative to their current position. Note that relative movements are usually more performant. A `RandomAccess` can be created by any object that implements `RandomAccessible`, like an `Img` or a `View`.

`Localizable` is implemented by `Cursor` as well as `RandomAccess`, which means they are able to report their current location. However, for `Cursor` we differentiate between a `LocalizingCursor` and a normal `Cursor`. A `LocalizingCursor` updates his position on every move, no matter if it is queried or not whereas a normal `Cursor` computes its location on demand. Using a `LocalizingCursor` is more efficient if the location is queried for every pixel, a `Cursor` will be faster when localizing only occasionally.

The `Sampler` interface implemented by `Cursor` and `RandomAccess` provides access to the `Type` instance of the current pixel. Using the `Type` instance it is possible to read and write its current value. Depending on the capabilities of the `Type` more operations are available, e.g. +,-,\*,/ if it is a `NumericType`.

Note that `IterableInterval` implements the java.lang.Iterable interface, which means it is compatible to specialized Java language constructs:

```java
// add 5 to every pixel
for ( UnsignedByteType type : img )
    type.add( 5 );
```

### Example 2a - Duplicating an Img using a generic method

The goal of this example is to make a copy of an existing `Img`. For this task it is sufficient to employ `Cursor`s. The order of iteration for both `Img`'s will be the same as they are instantiated using the same `ImgFactory`. It is possible to test if two `IterableInterval` have the same iteration order:

```java
boolean sameIterationOrder =
    interval1.iterationOrder().equals( interval2.iterationOrder() );
```

The copy method itself is a generic method, it will work on any kind of `Type`. In this particular case it works on a `FloatType`, but would also work on anything else like for example a `ComplexDoubleType`. The declaration of the generic type is done in the method declaration:

```java
public < T extends Type< T > > Img< T > copyImage( ... )
```
`< T extends Type< T > >` basically means that `T` can be anything that extends `Type`. These can be final implementations such as `FloatType` or also intermediate interfaces such as `RealType`. This, however, also means that in the method body only operations supported by `Type` will be available. Note that the method returns a `T`, which also means that in the constructor from which we call method it will also return an `Img<FloatType>` as we provide it with one.

{% include code org='imglib' repo='imglib2-tutorials' branch='master'
                        path='src/main/java/Example2a.java' line-start='34' %}

### Example 2b - Duplicating an Img using a different ImgFactory

<span style="color:#FF0000">**WARNING:** The `copyImageWrong` method in this example makes a mistake on purpose!</span> It intends to show that the iteration order of `Cursor`s is important to consider. The goal is to copy the content of an `ArrayImg` (i.e. an `Img` that was created using an `ArrayImgFactory`) into a `CellImg`. Using only `Cursor`s for both images will have a wrong result as an `ArrayImg` and a `CellImg` have different iteration orders. An `ArrayImg` is iterated linearly, while a `CellImg` is iterate cell-by-cell, but linearly within each cell.

<img src="/media/libs/imglib2/imglib2example2b.png" width="780"/> *Shows the result if two Cursors are used that have a different iteration order. Here we are wrongly copying an ArrayImg (left) into a CellImg (right).*

<span style="color:#FF0000">The correct code for the copy-method (in `copyImageCorrect`) requires the use of a `RandomAccess`.</span> We use a `Cursor` to iterate over all pixels of the input and a `RandomAccess` which we set to the same location the output. Note that the `setPosition()` call of the `RandomAccess` directly takes the `Cursor` as input, which is possible because `Cursor` implements `Localizable`. Please also note that we use a `LocalizingCursor` instead of a normal `Cursor` because we need the location of the `Cursor` at every pixel.

{% include code org='imglib' repo='imglib2-tutorials' branch='master'
                        path='src/main/java/Example2b.java' line-start='34' %}

### Example 2c - Generic copying of image data

In order to write a method that generically copies data requires an implementation for the underlying concepts of `RandomAccessible`, `Iterable` and `Interval`. In that way, it will run on `Img`, `View` and any other class implemented for these interfaces (even if they do not exist yet).

Therefore we design the copy method in a way that the *target* is an `IterableInterval` and the *source* is `RandomAccessible`. In this way, we simply iterate over the *target* and copy the corresponding pixels from the *source*.

As the *source* only needs to be `RandomAccessible`, it can be basically anything that can return a value at a certain location. This can be as simple as an `Img`, but also interpolated sparse data, a function, a ray-tracer, a `View`, ....

As the *target* needs to be an `IterableInterval`, it is more confined. This, however does not necessarily mean that it can only be an `Img` or a `View` that is not infinite. It simply means it has to be something that is iterable and not infinite, which for example also applies to sparse data (e.g. a list of locations and their values).

{% include code org='imglib' repo='imglib2-tutorials' branch='master'
                        path='src/main/java/Example2c.java' line-start='34' %}

## Example 3 - Writing generic algorithms

Examples 1 and 2 tried to introduce important tools you need in order to implement algorithms with ImgLib2. This example will show three generic implementations of algorithms computing the [min/max](#example-3a-min-max-search), average as well as the center of mass.

The core idea is to implement algorithms as generic as possible in order to maximize code-reusability. In general, a good way to start is to think: *What are the minimal requirements in order to implement algorithm X?* This applies to all of the following three concepts:

-   **Type:** You should always use the most abstract `Type` possible, i.e. the one that just offers enough operations to perform your goal. In this way, the algorithm will be able to run on `Type`s you might not even have thought about when implementing it. A good example is the min&max search in example 3a. Instead of implementing it for `FloatType` or the more abstract `RealType`, we implement it for the even more abstract `Comparable & Type`.
-   **Image data:** Every algorithm should only demand those interfaces that it requires, not specific implementations of it like `Img`. You might require `RandomAccessible` (infinite), `RandomAccessibleInterval` (finite), `Iterable` (values without location), `IterableInterval` (values and their location) or their corresponding interfaces for *real-valued locations* `RealRandomAccessible`, `RealRandomAccessibleRealInterval` and `IterableRealInterval`. Note that you can concatenate them if you need more than one property.
-   **Dimensionality:** Usually there is no reason to restrict an algorithm to a certain dimensionality (like only for two-dimensional images), at least we could not really come up with an convincing example  
    *If the application or plugin your are developing addresses a certain dimensionality (e.g. stitching of panorama photos) it is understandable that you do not want to implement everything n-dimensionally. But try to implement as many as possible of the smaller algorithm you are using as generic, n-dimensional methods. For example, everything that requires only to **iterate** the data is usually inherently n-dimensional.

Following those ideas, your newly implemented algorithm will be applicable to any kind of data and dimensionality it is defined for, not only a very small domain you are currently working with. Also note that quite often this actually makes the implementation simpler.

### Example 3a - Min/Max search

Searching for the minimal and maximal value in a dataset is a very nice example to illustrate generic algorithms. In order to find min/max values, `Type`s only need to be able to compare themselves. Therefore we do not need any numeric values, we only require them to implement the (Java) interface `Comparable`. Additionally, no random access to the data is required, we simply need to iterate all pixels, also their location is irrelevant. The image data we need only needs to be `Iterable`.

Below we show **three** small variations of the min/max search. [First](#example-3a-variation-1) we show the implementation as described above. [Second](#example-3a-variation-2) we illustrate that this also works on a standard Java `ArrayList`. [Third](#example-3a-variation-3) we show how the implementation changes if we do not only want the min/max value, but also their location. This requires to use `IterableInterval` instead, as `Cursor` can return their location.

#### Example 3a - Variation 1

{% include code org='imglib' repo='imglib2-tutorials' branch='master'
                        path='src/main/java/Example3a1.java' line-start='35' %}

#### Example 3a - Variation 2

Note that this example works just the same way if the input is not an `Img`, but for example just a standard Java `ArrayList`.

{% include code org='imglib' repo='imglib2-tutorials' branch='master'
                        path='src/main/java/Example3a2.java' line-start='34' %}

#### Example 3a - Variation 3

If we want to compute the location of the minimal and maximal pixel value, an `Iterator` will not be sufficient as we need location information. Instead the location search will demand an `IterableInterval` as input data which can create `Cursor`s. Apart from that, the algorithm looks quite similar. Note that we do not use a `LocalizingCursor` but only a `Cursor` the location happens only when a new maximal or minimal value has been found while iterating the data.

{% include code org='imglib' repo='imglib2-tutorials' branch='master'
                        path='src/main/java/Example3a3.java' line-start='34' %}

### Example 3b - Computing average

In a very similar way one can compute the average intensity for image data. Note that we restrict the `Type` of data to `RealType`. In theory, we could use `NumericType` as it offers the possibility to *add* up values. However, we cannot ensure that `NumericType` provided is capable of adding up millions of pixels without overflow. And even if we would ask for a second `NumericType` that is capable of adding values up, it might still have numerical instabilities. *Note that actually every Java native type has those instabilities*. Therefore we use the `RealSum` class that offers correct addition of even very large amounts of pixels. As this implementation is only available for double values, we restrict the method here to `RealType`.

{% include code org='imglib' repo='imglib2-tutorials' branch='master'
                        path='src/main/java/Example3b.java' line-start='34' %}

## Example 4 - Specialized iterables

Example 4 will focus on how to work with specialized **iterables**. They are especially useful when performing operations in the local neighborhood of many pixels - like finding local minima/maxima, texture analysis, convolution with non-separable, non-linear filters and many more. One elegant solution is to write a specialized `Iterable` that will iterate all pixels in the local neighborhood. We implemented two examples:

-   A `HyperSphere` that will iterate a n-dimensional sphere with a given radius at a defined location.
-   A `LocalNeighborhood` that will iterate n-dimensionally all pixels adjacent to a certain location, but skip the central pixel (this corresponds to an both neighbors in 1d, an 8-neighborhood in 2d, a 26-neighborhood in 3d, and so on ...)

### Example 4a - Drawing a sphere full of spheres

In the first sample we simply draw a sphere full of little spheres. We therefore create a large `HyperSphere` in the center of a `RandomAccessibleInterval`. Note that the `HyperSphere` only needs a `RandomAccessible`, we need the additional `Interval` simply to compute the center and the radius of the large sphere. When iterating over all pixels of this large sphere, we create small `HyperSphere`s at every n'th pixel and fill them with a random intensity.

This example illustrates the use of specialized `Iterable`s, and emphasizes the fact that they can be stacked on the underlying `RandomAccessible` using the location of one as the center of a new one. Note that we always create new instances of `HyperSphere`. The code reads very nicely but might not offer the best performance. We therefore added *update* methods to the `HyperSphere` and its `Cursor` that could be used instead.

Another interesting aspect of this example is the use of the `ImagePlusImgFactory`, which is the compatibility container for ImageJ. If the required dimensionality and `Type` is available in ImageJ, it will internally create an ImagePlus and work on it directly. In this case, one can request the ImagePlus and show it directly. It will, however, fail if `Type` and dimensionality is not supported by ImageJ and throw a `ImgLibException`.

<img src="/media/libs/imglib2/manyspheres.jpg" width="780"/> *Shows the result of example 4a for the (a) two-dimensional, (b) three-dimensional and (c) four-dimensional case. The image series in (c) represents a movie of a three-dimensional rendering. The images of (b) and (c) were rendered using the ImageJ 3d Viewer.*

{% include code org='imglib' repo='imglib2-tutorials' branch='master'
                        path='src/main/java/Example4a.java' line-start='34' %}

### Example 4b - Finding and displaying local minima

In this example we want to find all local minima in an image an display them as small spheres. To not capture too much of the noise in the image data, we first perform an in-place Gaussian smoothing with a sigma of 1, i.e. the data will be overwritten with the result. A complete documentation of the gauss package for ImgLib2 can be found [here](Gauss_Package_ImgLib2).

We display the results using a binary image. *Note that the `BitType` only requires one bit per pixel and therefore is very memory efficient.*

The generic method for minima detection has some more interesting properties.
The type of the source image data actually does not require to be of `Type`,
it simply needs something that is comparable. The `LocalNeighborhood` will
iterate n-dimensionally all pixels adjacent to a certain location, but skip the
central pixel (this corresponds to an both neighbors in 1d, an 8-neighborhood
in 2d, a 26-neighborhood in 3d, and so on ...). This allows to efficiently
detect if a pixel is a local minima or maxima. Note that the `Cursor` that
performs the iteration can have special implementations for specific
dimensionalities to speed up the iteration. See below the example for a
specialized three-dimensional iteration:

Access plan for a 3d neighborhood, starting at the center position marked by
(x). The initial position is, in this example, NOT part of iteration, which
means the center pixel is not iterated. Note that every step except for the
last one can be done with a very simple move command.
  
```
upper z plane (z-1)    center z plane (z=0)    lower z plane(z+1)
-------------          -------------           -------------
| 2 | 1 | 8 |          | 11| 10| 9 |           | 20| 19| 18|
|------------          -------------           -------------
| 3 | 0 | 7 |          | 12| x | 16|           | 21| 25| 17|
|------------          -------------           -------------
| 4 | 5 | 6 |          | 13| 14| 15|           | 22| 23| 24|
-------------          -------------           -------------
```

Please note as well that if one would increase the radius of the `RectangleShape` to more than 1 (without at the same time changing the `View` on source that creates an inset border of exactly this one pixel), this example would fail as we would try to write image data outside of the defined boundary. `OutOfBoundsStrategies` which define how to handle such cases is discussed in example 5.

<img src="/media/libs/imglib2/allminima.jpg" width="780"/> *Shows the result of the detection of local minima after the Gaussian blurring. (a) depicts the input image, (b) the blurred version (sigma=1) and (c) all local mimina drawn as circles with radius 1.*

{% include code org='imglib' repo='imglib2-tutorials' branch='master'
                        path='src/main/java/Example4b.java' line-start='34' %}

## Example 5 - Out of bounds

Many algorithms like *convolutions* require to access pixels outside of an `Interval`, i.e. also pixels outside of an image. In ImgLib2 this is handled using `Views` which convert a `RandomAccessibleInterval` into an infinite `RandomAccessible` using an `OutOfBoundsStrategy`. Those infinite `RandomAccessible`s are able to return pixel values at any arbitrary location.

*Important: One should never access pixels outside of the defined `Interval` as it will in most cases result in unexpected behavior, depending on the kind of underlying `RandomAccessible`. If it is for example an `Img`, it will return wrong values or throw an exception.*

Which `OutOfBoundsStrategy` to use depends on task you want to perform. For convolutions we suggest the mirror strategy as it introduces the least artifacts. When working on Fourier images, the periodic strategy applies best as it correctly mimics its spatial properties. Random Value strategies might be useful to avoid accidental correlations and constant value strategies are the most performant and might work well for simple operations or to avoid exceptions when accidental writing or reading outside of the Interval occurs.

<img src="/media/outofbounds.jpg" width="780"/> *Illustrates the effect of various OutOfBoundsStrategies. (a) shows out of bounds with a constant value, (b) shows a mirroring strategy, (c) shows the periodic strategy, and (d) shows a strategy that uses random values.*

{% include code org='imglib' repo='imglib2-tutorials' branch='master'
                        path='src/main/java/Example5.java' line-start='34' %}

## Example 6 - Basic built-in algorithms

ImgLib2 contains a growing number of built-in standard algorithms. In this section, we will show some of those, illustrate how to use them and give some examples of what it might be used for.

Typically algorithms provide static methods for simple calling, but they also have classes which you can instantiate yourself to have more options.

*Important: Algorithms do not allow to work on a different dimensionality than the input data. You can achieve that by selecting hyperslices using Views (see Example 6a - version 4). In this way you can for example apply two-dimensional gaussians to each frame of a movie independently.*

### Example 6a - Gaussian convolution

The Gaussian convolution has its own [wiki page](Gauss_Package_ImgLib2). You can apply the Gaussian convolution with different sigmas in any dimension. It will work on any kind `RandomAccessibleInterval`. Below we show a examples of a simple gaussian convolution (variation 1), convolution using a different `OutOfBoundsStrategy` (variation 2), convolution of a part of an `Interval` (variation 3), and convolution of in a lower dimensionality than the image data (variation 4).

<img src="/media/libs/imglib2/gaussexamples.jpg" width="780"/> *Shows the result of the four examples for Gaussian convolution. (a) shows a simple Gaussian convolution with sigma=8. (b) shows the same Gaussian convolution but using an OutOfBoundsConstantValue instead. (c) shows the result when convolving part of the image in-place. (d) shows the result when individually convolving 1-dimensional parts on the image.*

#### Example 6a - Gaussian convolution (variation 1 - simple)

Here, we simply apply a Gaussian convolution with a sigma of 8. Note that it could be applied in-place as well when calling *Gauss.inFloatInPlace( ... )*. The Gaussian convolution uses by default the `OutOfBoundsMirrorStrategy`.

{% include code org='imglib' repo='imglib2-tutorials' branch='master'
                        path='src/main/java/Example6a1.java' line-start='34' %}

#### Example 6a - Gaussian convolution (variation 2 - different OutOfBoundsStrategy)

Here we use an `OutOfBoundsStrategyConstantValue` instead. It results in continuously darker borders as the zero-values from outside of the image are used in the convolution. Note that the computation is done in-place here. However, we still need to provide an `ImgFactory` as the Gaussian convolution needs to create temporary image(s) - except for the one-dimensional case.

{% include code org='imglib' repo='imglib2-tutorials' branch='master'
                        path='src/main/java/Example6a2.java' line-start='34' %}

#### Example 6a - Gaussian convolution (variation 3 - only part of an Interval)

Here we only convolve part of an `Interval`, or in this case part of the `Img`. Note that for convolution he will actually use the real image values outside of the defined `Interval`. The `OutOfBoundsStrategy` is only necessary if the kernel is that large so that it will actually grep image values outside of the underlying `Img`.

Note: if you wanted, you could force him to use an `OutOfBoundsStrategy` directly outside of the `Interval`. For that you would have to create an `RandomAccessibleInterval` on the `Img`, extend it by an `OutOfBoundsStrategy` and give this as input to the Gaussian convolution.

{% include code org='imglib' repo='imglib2-tutorials' branch='master'
                        path='src/main/java/Example6a3.java' line-start='34' %}

#### Example 6a - Gaussian convolution (variation 4 - with a lower dimensionality)

This example shows howto apply an algorithm to a lower dimensionality as the image data you are working on. Therefore we use `Views` to create HyperSlices which have *n-1* dimensions. We simply apply the algorithm in-place on those `Views` which will automatically update the image data in the higher-dimensional data.

Specifically, we apply 1-dimensional Gaussian convolution in 30-pixel wide stripes using a sigma of 16. *Note that whenever you request an `HyperSlice` for a certain dimension, you will get back a `View` that contains all dimensions <span style="color:#FF0000">but</span> this one.*

{% include code org='imglib' repo='imglib2-tutorials' branch='master'
                        path='src/main/java/Example6a4.java' line-start='34' %}

### Example 6b - Convolution in Fourier space

In image processing it is sometimes necessary to convolve images with non-separable kernels. This can be efficiently done in Fourier space exploiting the *convolution theorem*. It states that a convolution in real-space corresponds to a multiplication in Fourier-space, as vice versa. Note that the computation time for such a convolution is independent of the size and shape of the kernel.

*Note that it is useful to normalize the kernel prior to Fourier convolution so that the sum of all pixels is one. Otherwise, the resulting intensities will be increased.*

<img src="/media/libs/imglib2/fourierconvolution.jpg" width="780"/> *Shows the effect of the Fourier convolution. The left image was convolved with the kernel depicted in the lower left corner, the right panel shows the convolved image. Note that the computation speed does not depend on the size or the shape of the kernel.*

<span style="color:#FF0000">*Important: This source code is only GPLv2!*</span>

{% include code org='imglib' repo='imglib2-tutorials' branch='master'
                        path='src/main/java/Example6b.java' line-start='54' %}

### Example 6c - Complex numbers and Fourier transforms

In this example we show how to work with complex numbers and Fourier transforms. We show how to determine the location of a template in an image exploiting the *Fourier Shift Theorem*. We therefore compute the Fast Fourier Transform of a template, invert it and convolve it in Fourier space with the original image.

Computing an FFT is straight forward. It does not offer a static method because the instance of the FFT is required to perform an inverse FFT. This is necessary because the input image needs to be extended to a size supported by the 1-d FFT method (`edu_mines_jtk.jar`). In order to restore the complete input image remembering those parameters is essential.

For the display of complex image data we provide `Converter`s to display the [gLog](http://bioinformatics.oxfordjournals.org/content/19/8/966) of the power-spectrum (default), phase-spectrum, real values, and imaginary values. It is, however, straight forward to implement you own `Converter`s.

Note that for inverting the kernel we use methods defined for `ComplexType`, also the basic math operations add, mul, sub and div are implemented in complex math. The inverse FFT finally takes the instance of the FFT as a parameter from which it takes all required parameters for a correct inversion.

The final convolution of the inverse template with the image is performed using the FourierConvolution (see example 6b). Note that all possible locations of the template in the image have been tested. The peak in the result image clearly marks the location of the template, while the computation time for the whole operation takes less than a second.

<img src="/media/fourier.jpg" width="780"/> *Shows the result and intermediate steps of the template matching using the Fourier space. In the upper panel you see the input image as well as the template that we use from matching. Below we show four different views of the Fast Fourier Transform of the template: the power spectrum, the phase spectrum, the real values, and the imaginary values. In the lower panel you see the result of the convolution of the inverse template with the image. The position where the template was located in the image is significantly visible. In the bottom right corner you see the inverse FFT of the inverse kernel.*

<span style="color:#FF0000">*Important: This source code is only GPLv2!*</span>

{% include code org='imglib' repo='imglib2-tutorials' branch='master'
                        path='src/main/java/Example6c.java' line-start='54' %}

## Example 7 - Interpolation

Interpolation is a basic operation required in many image processing tasks. In the terminology of ImgLib2 it means to convert a `RandomAccessible` into a `RealRandomAccessible` which is able to create a `RealRandomAccess`. It can be positioned at real coordinates instead of only integer coordinates and a return a value for each real location. Currently, three interpolation schemes are available for ImgLib2:

-   **Nearest neighbor interpolation** (also for available for any kind of data that can return a nearest neighbor like sparse datasets)
-   **Linear interpolation**
-   **Lanczos interpolation**

In the example we magnify a given real interval in the `RealRandomAccessible` which is based on the interpolation on an `Img` and compare the results of all three interpolation methods.

<img src="/media/interpolation.jpg" width="779"/> *Shows the result for three different interpolators when magnifying a small part of the image by 10x. The nearest neighbor interpolation is computed fastest and is the most versatile as it requires no computation but just a lookout. The result is, however, very pixelated. The linear interpolation produces reasonable results and computes quite fast. The Lanczos interpolation shows visually most pleasing results but also introduces slight artifacts in the background.*

{% include code org='imglib' repo='imglib2-tutorials' branch='master'
                        path='src/main/java/Example7.java' line-start='34' %}

## Example 8 - Working with sparse data

ImgLib2 supports sparsely sampled data, i.e. collections of locations together with their value. Such datasets typically implement the `IterableRealInterval` interface, which means they can be iterated and have real-valued locations in n-dimensional space. Currently ImgLib2 supports to store such collections either as list (`RealPointSampleList`) or `KDTree`. The `RealPointSampleList` can be iterated, whereas the `KDTree` additionally supports three efficient ways of searching for nearest neighboring points in the n-dimensional space (`NearestNeighborSearch`, `KNearestNeighborSearch`, and `RadiusNeighborSearch`).

In order to display sparse data ImgLib2 currently supports two interpolation schemes, the `NearestNeighborInterpolation` and the `InverseDistanceWeightingInterpolation`. They can compute a value for every location in space by returning either the value of the closest sample or an interpolated, distance-weighted value of the *k* nearest neighbors to the sampled location. The interpolation scheme therefore converts any `IterableRealInterval` into a `RealRandomAccessible` that can be displayed by wrapping it into a `RandomAccessible` and defining `Interval` using `Views`.

*This is, however, not only useful for display. Note that you execute on such data any algorithm or method that is implemented for `RealRandomAccessible` or `RandomAccessible`, like Gaussian convolution in the first example. <span style="color:#FF0000">Note that none of these ever exists in memory, it is done completely virtual on just the sparse samples.</span>*

### Example 8a - Create random sparse data, display and convolve it

In this example we create a certain number of random samples with random intensities inside a certain `Interval`. Using nearest neighbor interpolation we wrap it into a `RealRandomAccessible`, wrap it again into a `RandomAccessible`, define an `Interval` on it and display it. On the same *virtual* data we perform a Gaussian convolution and show it, too.

<img src="/media/libs/imglib2/sparseimage1.jpg" width="777"/> *On the left hand side it shows nearest-neighbor rendered random sparse data as created in example 8a. The right hand side shows the result of a Gaussian convolution, run directly on the virtual RandomAccessibleInterval.*

{% include code org='imglib' repo='imglib2-tutorials' branch='master'
                        path='src/main/java/Example8a.java' line-start='34' %}

### Example 8b - Randomly sample an existing image and display it

In this example we sample an existing image at random locations and render the result using a nearest neighbor interpolation as well as a distance-weighted average of the k nearest neighbors.

<img src="/media/libs/imglib2/sparseimage2.jpg" width="638"/>  
*Shows the result of sparse sampling of an existing image using a varying number of random samples. The upper panel shows the rendering using nearest neighbor interpolation, the lower panel uses an interpolated, distance-weighted value of the k nearest neighbors relative to each sampled location (i.e. each pixel).*

{% include code org='imglib' repo='imglib2-tutorials' branch='master'
                        path='src/main/java/Example8b.java' line-start='34' %}

# RNifti: Fast R and C++ Access to NIfTI Images

The [NIfTI-1 format](http://nifti.nimh.nih.gov/nifti-1) is a popular file format for storing medical imaging data, widely used in medical research and related fields. Conceptually, a NIfTI-1 file incorporates multidimensional numeric data, like an R `array`, but with additional metadata describing the real-space resolution of the image, the physical orientation of the image, and how the image should be interpreted.

There are several packages available for reading and writing NIfTI-1 files in R, and these are summarised in the [Medical Imaging task view](https://cran.r-project.org/web/views/MedicalImaging.html). However, `RNifti` is distinguished by its

- [extremely strong performance](#performance), in terms of speed;
- [C/C++ API](#api), allowing access to NIfTI images even in compiled code in other packages; and
- modest dependencies, consisting of only R itself and the very widely-used [Rcpp](https://cran.r-project.org/package=Rcpp) C++ wrapper library.

The latest development version of the package can always been installed from GitHub using the `devtools` package.

```r
## install.packages("devtools")
devtools::install_github("jonclayden/RNifti")
```

## Usage

The primary role of `RNifti` is in reading and writing NIfTI-1 files, either `gzip`-compressed or uncompressed, and providing access to image data and metadata. An image may be read into R using the `readNifti` function.

```r
library(RNifti)
image <- readNifti(system.file("extdata", "example.nii.gz", package="RNifti"))
```

This image is an R array with some additional attributes containing information such as its dimensions and the size of its pixels (or voxels, in this case, since it is a 3D image). There are auxiliary functions for extracting this information: the standard `dim()`, plus `pixdim()` and `pixunits()`.

```r
dim(image)
# [1] 96 96 60

pixdim(image)
# [1] 2.5 2.5 2.5

pixunits(image)
# [1] "mm" "s"
```

So this image is of size 96 x 96 x 60 voxels, with each voxel representing 2.5 x 2.5 x 2.5 mm in real space. (The temporal unit, seconds here, only applies to the fourth dimension, if it is present.) Replacement versions of the latter functions are also available, for modifying the metadata.

A fuller list of the raw metadata stored in the file can be obtained using the `dumpNifti` function.

```r
dumpNifti(image)
# NIfTI-1 header
#     sizeof_hdr: 348
#       dim_info: 0
#            dim: 3  96  96  60  1  1  1  1
#      intent_p1: 0
#      intent_p2: 0
#      intent_p3: 0
#    intent_code: 0
#       datatype: 64
#         bitpix: 64
#    slice_start: 0
#         pixdim: -1.0  2.5  2.5  2.5  0.0  0.0  0.0  0.0
#     vox_offset: 352
#      scl_slope: 1
#      scl_inter: 0
#      slice_end: 0
#     slice_code: 0
#     xyzt_units: 10
#        cal_max: 2503
#        cal_min: 0
# slice_duration: 0
#        toffset: 0
#        descrip: FSL5.0
#       aux_file: 
#     qform_code: 2
#     sform_code: 2
#      quatern_b: 0
#      quatern_c: 1
#      quatern_d: 0
#      qoffset_x: 122.0339
#      qoffset_y: -95.18523
#      qoffset_z: -55.03814
#         srow_x: -2.5000  0.0000  0.0000  122.0339
#         srow_y: 0.00000  2.50000  0.00000  -95.18523
#         srow_z: 0.00000  0.00000  2.50000  -55.03814
#    intent_name: 
#          magic: n+1
```

Advanced users who know the NIfTI format well may want to alter elements of this metadata directly, and the `updateNifti` function provides a mechanism for this, either by passing a second, "template" image, or by providing lists with elements named as above, as in

```r
image <- updateNifti(image, list(intent_code=1L))
```

An image can be written back to NIfTI-1 format using the `writeNifti` function.

```r
writeNifti(image, "file.nii.gz")
```

## Performance

The `RNifti` package uses the robust NIfTI-1 reference implementation, which is written in C, to read and write NIfTI files. It also uses the standard NIfTI-1 data structure as its canonical representation of a file in memory. Together, these make the package extremely fast, as the following benchmark against packages [`AnalyzeFMRI`](https://cran.r-project.org/package=AnalyzeFMRI), [`neuroim`](https://cran.r-project.org/package=neuroim), [`oro.nifti`](https://cran.r-project.org/package=oro.nifti) and [`tractor.base`](https://cran.r-project.org/package=tractor.base) shows.

```r
installed.packages()[c("AnalyzeFMRI","neuroim","oro.nifti","RNifti",
                       "tractor.base"), "Version"]
#  AnalyzeFMRI      neuroim    oro.nifti       RNifti tractor.base 
#     "1.1-16"      "0.0.6"    "0.5.5.2"      "0.1.0"      "3.0.5" 

library(microbenchmark)
microbenchmark(AnalyzeFMRI::f.read.volume("example.nii"),
               neuroim::loadVolume("example.nii"),
               oro.nifti::readNIfTI("example.nii"),
               RNifti::readNifti("example.nii"),
               tractor.base::readImageFile("example.nii"))
# Output level is not set; defaulting to "Info"
# Unit: milliseconds
#                                        expr       min        lq      mean
#   AnalyzeFMRI::f.read.volume("example.nii") 35.257563 42.604118  66.32578
#          neuroim::loadVolume("example.nii") 52.529019 61.301876 130.82085
#         oro.nifti::readNIfTI("example.nii") 79.877138 92.201510 174.57330
#            RNifti::readNifti("example.nii")  3.004172  3.702351  14.00659
#  tractor.base::readImageFile("example.nii") 45.307781 51.006998  61.80458
#      median         uq       max neval
#   45.303426  50.106693  556.8474   100
#   66.444186 192.282696 2580.7596   100
#  215.847429 224.826626  712.8495   100
#    4.472429   7.061472  168.5171   100
#   54.334831  57.776505  216.4457   100
```

With a median runtime of just over 4 ms, `RNifti` is typically ten times as fast as any of the alternatives to read an image into R.

## API

*This section will only be of interest to package developers who may want to work with NIfTI images in their compiled code. And perhaps the very curious!*

The package does not fully duplicate the NIfTI-1 structure's contents in R-visible objects. Instead, it passes key metadata back to R, such as the image dimensions and pixel dimensions, and it also passes back the pixel values where they are needed. It also creates an [external pointer](http://r-manuals.flakery.org/R-exts.html#External-pointers-and-weak-references) to the native data structure, which is stored in an attribute. This pointer is dereferenced whenever the object is passed back to the C++ code, thereby avoiding unnecessary duplication and ensuring that all metadata remains intact. The full NIfTI-1 header can be obtained using the `dumpNifti` R function, if it is needed.

This arrangement is efficient and generally works well, but many R operations strip attributes—in which case the external pointer will be removed. The internal structure will be built again when necessary, but using default metadata. In these cases, if it is important to keep the original metadata, the `updateNifti` function should be called explicitly, with a template object. This reconstructs the NIfTI-1 data structure, using the template as a starting point.

It is also possible to use the package's NIfTI-handling code in other R packages' compiled code, thereby obviating the need to duplicate the reference implementation. Moreover, `RNifti` provides a C++ wrapper class, `NiftiImage`, which simplifies memory management, supports the package's internal image pointers and persistence, and provides syntactic sugar. A package can use this class by including

```
LinkingTo: RNifti
```

in its `DESCRIPTION` file, and then including the `RNifti.h` header file. For example,

```c++
#include "RNifti.h"

void myfunction ()
{
    NiftiImage image("example.nii.gz");
    // Do something with the image
}
```

There are also constructors taking a `SEXP`, another `NiftiImage`, or a `nifti_image` structure from the reference implementation. `NiftiImage` objects can be implicitly cast to pointers to `nifti_image` structs, meaning that they can be directly used in calls to the reference implementation's own API. The latter is accessed through the separate `RNiftiAPI.h` header file.

```c++
#include "RNifti.h"
#include "RNiftiAPI.h"

void myfunction (SEXP image_)
{
    NiftiImage image(image_);
    const size_t volsize = nifti_get_volsize(image);
}
```

(`RNifti` will also have to be added to the import list in the package's `DESCRIPTION` file, as well as `LinkingTo`.) The `RNiftiAPI.h` header should only be included once per package, since it contains function implementations. Multiple includes will lead to duplicate symbol warnings from your linker. Therefore, if multiple source files require access to the NIfTI-1 reference implementation, it is recommended that the API header be included alone in a separate ".c" or ".cpp" file, while others only include the main `RNifti.h`.

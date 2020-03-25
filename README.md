# Regtools

Tools for manipulating, combining and applying image transformations.  

## Overview

The following three classes are provided for working with registrations, motion corrections and image spaces. 

`Registration`: a 4x4 affine transformation, that optionally can be associated with a specific source and reference image ('from' and 'to'). Internally, all registrations are stored in world-world terms, and all interactions between registrations are also in world-world terms unless expressly requested. 

`MotionCorrection`: a sequence of `Registration` objects, one for each volume of a timeseries. 

`ImageSpace`: the voxel grid of an image, including the dimensions, voxel size and orientation (almost everything except the image itself). This class also allows easy manipulation of the grid (shifting, cropping, resizing voxels, etc)

## Loading, converting and saving

`Registration` objects can be initialised from a text file or `np.array`. If the registration was produced by FLIRT, paths to the source and reference images are required to convert the transformation. 

```python  
src = 'source_image.nii.gz'
ref = 'reference_image.nii.gz'

# A simple array is assumed to be in world-world terms 
r1 = Registration(an_array)
# Convert to FSL, returns a np.array
r1.to_fsl(src, ref) 
# Save as FSL
r1.save_txt('r1_fsl.txt', src, ref, 'fsl') 

# If the src and ref are provided, FSL/FLIRT convention is assumed
# The conversion to world-world terms is automatic 
r2 = Registration('a_matrix.txt', src, ref)
# Return inverse Registration
r2.inverse() 
# Inverse FSL transform as np.array 
r2.inverse().to_fsl(ref, src) 
# Save as text file 
r2.save_txt('r2_world.txt')
# Save inverse as FSL
r2.inverse().save_txt('r2_inv_fsl.txt', ref, src, 'fsl') 

# Imagine we want to apply the transformation represented by r2, 
# but keep the result within the same voxel grid. In FSL terms: 
r2.save_txt('r2_fsl_samespace.txt', src, src, 'fsl')
```

`MotionCorrection` objects can be initialised from a directory containing transformations, a list of text file paths, or a list of `np.array`s. Once again, if the registration was produced by MCFLIRT, paths to the source and reference images are required to convert the transformation. The convention will be assumed as with `Registration`, or you can state it explicitly. 

```python
m1 = MotionCorrection('mcflirt_directory', src) # load from MCFLIRT directory
m1.save_txt('world_directory') # save as text files, world-world
m2 = MotionCorrection(list_of_arrays) # create from list of arrays
```

`ImageSpace` objects can be initialised with a nibabel Nifti object or a path to Nifti image. 
```python
src_spc = ImageSpace(src_nifti) # from a nibabel nifti 
ref_spc = ImageSpace('ref.nii.gz') # from a path 
ref_spc.save_image(some_array, 'array_in_ref.nii.gz') # save some data in this space 
```

## Combining transformations

Transformations may be combined my matrix multiplication. `Registration`s, `MotionCorrection`s and `np.array`s can all be combined in this manner. Note that the result of a multiplication with a `np.array` will be a `Registration` object. When multiplying a `MotionCorrection` and a `Registration`, the result will be a new `MotionCorrection` object. The order of multiplication is important: to apply the transformation A then B, the matrix multiplication `B @ A` should be used. The safest way of combining registrations is to use the `chain()` function - it works on any number of transforms and takes care of the order for you!

```python
# Three images (A,B,C), and three transformations: A->B, motion correction for A
# and C->B. 
a2a_moco = MotionCorrection('a_mcflirt_directory', 'a.nii.gz')
a2b = Registration('a2b.txt')
c2b = Registration('b2c_flirt.mat', 'c.nii.gz', 'b.nii.gz')

# Get a single transformation for A->C, including motion correction 
# NB the result will be promoted to a MotionCorrection object 
a2c_moco = chain(a2a_moco, a2b, c2b.inverse())

# Alternatively, do the multiplication directly: 
a2c_moco_2 = (c2b.inverse() @ a2b @ a2a_moco)

# Save as world-world matrices, in FSL convention
a2c_moco.save_txt('a2c_moco_dir', 'a.nii.gz', 'c.nii.gz')
```

## Applying transformations 

Both `Registration`s and `MotionCorrection`s may applied with the `apply_to()` method. This uses SciPy's `ndimage.interpolation.map_coordinates()` function under the hood, permitting spline interpolation from order 1 (trilinear) to 5 (quintic) with pre-filtering to reduce interpolation artefacts. All `**kwargs` accepted by `map_coordinates()` may be passed to `apply_to()`. 

```python
a_img_3D = 'some_volume.nii.gz'
b_img_4D = 'some_timeseries.nii.gz'
a2b = Regisration('a2b.txt')
b2b_moco = MotionCorrection('b_mcflirt_dir', b_img_4D)

# A single registration can be applied to both 3D and 4D data
a2b.apply_to(a_img_3D, b_img_4D) # map a onto b, return nibabel nifti
# transform b onto a, save the result as a nifti 
a2b.inverse().apply_to(b_img_4D, a_img3D, out='timeseries_on_a.nii.gz')

# Chain the motion correction and transform b2a together 
# MotionCorrections can only be applied to 4D data 
b2a_moco = chain(b2b_moco, a2b.inverse())
b2a_moco.apply_to(b_img_4D, a_img_3D, out='timeseries_on_a_mc.nii.gz')

# Apply the chained transformation, but without resampling the result onto
# the voxel grid of b_img_3D (keep it in the original space of the timeseries)
b2a_moco.apply_to(b_img_4D, b_img_4D, out='timeseries_on_a_in_b_mc.nii.gz')
```

## More examples

to come 

## Further reading
An explanation of the FSL coordinate system: https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/FLIRT/FAQ
# wb-cifti-tutorial
Connectome workbench and GIFTI/CIFTI tutorial 

Slides:
https://docs.google.com/presentation/d/1aE4J3XDQ8pWLJfAehLS7NBeiLoyq4SzpgZHUf81OkvY/edit?usp=sharing

# Hands-on tutorial material will be updated here


## Example: myelin mapping

Prerequisites:
- BIDS dataset with T1w and T2w data
- T2w registered and resampled to T1w (e.g. itksnap, ants, greedy)
- fmriprep processed data (including freesurfer)


1. do math on volume space to get a ratio
```
wb_command -volume-math "clamp(T1w/T2w,0,10)" sub-0001_T1wDividedByT2w.nii.gz -var T1w ../data/bids/sub-0001/anat/sub-0001_acq-MPRAGE_run-01_T1w.nii.gz  -var T2w ../data/bids/sub-0001/anat/sub-0001_acq-SPACE_run-01_desc-regToT1w_T2w.nii.gz 
```

2. sample this on the midsurface
```
wb_command -volume-to-surface-mapping sub-0001_T1wDividedByT2w.nii.gz ../data/fmriprep_20.2.1/sub-0001/anat/sub-0001_acq-MPRAGE_run-1_hemi-L_midthickness.surf.gii sub-0001_hemi-L_myelin.func.gii -trilinear

wb_command -volume-to-surface-mapping sub-0001_T1wDividedByT2w.nii.gz ../data/fmriprep_20.2.1/sub-0001/anat/sub-0001_acq-MPRAGE_run-1_hemi-R_midthickness.surf.gii sub-0001_hemi-R_myelin.func.gii -trilinear
```

3. visualize in `wb_view`

4. some other things you could do here:

optional: get clusters of high myelin
```
wb_command -metric-find-clusters sub-0001_acq-MPRAGE_run-1_hemi-L_midthickness.surf.gii sub-0001_hemi-L_smooth-3mm_myelin.func.gii 2.5 50 sub-0001_hemi-L_smooth-3mm_myelinclusters.func.gii
```

optional: calculate a gradient 
```
wb_command -metric-gradient ?h.midthickness.surf.gii ?h.myelin.func.gii ?h.myelingradient.func.gii
```

Set range for color mapping to metric files
```
wb_command -metric-palette ?h.myelin.func.gii MODE_USER_SCALE -pos-user 0 0.05 -interpolate true -palette-name videen_style -disp-pos true -disp-neg false -disp-zero false
```




## Templates

Now, what if we want to use an atlas parcellation or visualize on a standard template??

###  parcellations
 - HCP-MMP
https://balsa.wustl.edu/reference/show/6V6gD
 - Schaeffer 
 https://github.com/ThomasYeoLab/CBIG/tree/master/stable_projects/brain_parcellation/Schaefer2018_LocalGlobal/Parcellations/HCP/fslr32k/cifti



1. We need to register between a subject and a template. Thankfully Freesurfer does this already (other common method is MSM - multimodal surface mapping). 

The registration output is the vertices of the subject surface, moved onto a sphere.
This is `surf/lh.sphere.reg` in Freesurfer's output. This is registered to a surface template called `fsaverage`. The other common template is `fsLR`. 

We can first convert this to `gifti` using 
```
mris_convert data/freesurfer_7.1/sub-0001/surf/lh.sphere.reg myelin_mapping/sub-0001_acq-MPRAGE_run-1_hemi-L_sphere.surf.gii
```

Now, this has a surface geometry (# of vertices, or density) that is specific to that subject, and is different for each subject. But since it is aligned in the sphere, we can use it to resample things (surface, metrics, labels) to another geometry.

1. resampling subject surfaces to fsLR

This resamples the subject midthickness surface to the `fsLR` geometry (note: here the `fsLR` is in `fsaverage` space - turns out this is just a simple rotation of the sphere, but we don't have to worry about it, we just need to make sure we use the `fsLR` sphere that is in `space-fsaverage`)

```
wb_command -surface-resample sub-0001_acq-MPRAGE_run-1_hemi-R_midthickness.surf.gii  sub-0001_acq-MPRAGE_run-1_hemi-R_sphere.surf.gii  ../data/templateflow/tpl-fsLR/tpl-fsLR_space-fsaverage_hemi-R_den-32k_sphere.surf.gii BARYCENTRIC sub-0001_acq-MPRAGE_run-1_hemi-R_space-fsLR_den-32k_midthickness.surf.gii

wb_command -surface-resample sub-0001_acq-MPRAGE_run-1_hemi-L_midthickness.surf.gii  sub-0001_acq-MPRAGE_run-1_hemi-L_sphere.surf.gii  ../data/templateflow/tpl-fsLR/tpl-fsLR_space-fsaverage_hemi-L_den-32k_sphere.surf.gii BARYCENTRIC sub-0001_acq-MPRAGE_run-1_hemi-L_space-fsLR_den-32k_midthickness.surf.gii
```

Now you can try visualizing standard 32k data on the subject in `wb_view`.


2. resampling subject metrics
```
wb_command -metric-resample sub-0001_hemi-L_smooth-3mm_myelin.func.gii sub-0001_acq-MPRAGE_run-1_hemi-L_sphere.surf.gii ../data/templateflow/tpl-fsLR/tpl-fsLR_space-fsaverage_hemi-L_den-32k_sphere.surf.gii  ADAP_BARY_AREA  sub-0001_hemi-L_smooth-3mm_space-fsLR_den-32k_myelin.func.gii -area-surfs sub-0001_acq-MPRAGE_run-1_hemi-L_midthickness.surf.gii sub-0001_acq-MPRAGE_run-1_hemi-L_space-fsLR_den-32k_midthickness.surf.gii
```

## CIFTI

In addition to the surface-based processing above, the combined surf+vol ("Grayordinates") are another reason to work with CIFTI.

The dense domain is made up of l/r cortical surfaces, and subcortical/cerebellar GM regions



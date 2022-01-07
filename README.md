# Clinical Applications of Quantitative MRI in Healthy Adolescents - Project Pipeline

Pipeline &amp; Research Notebook - Neuroscience MSc Research Project in "Clinical Applications of Quantitative MRI in Healthy Adolescents"

This Source Code Form is subject to the terms of the Mozilla Public License, v. 2.0. If a copy of the MPL was not distributed with this file, You can obtain one at http://mozilla.org/MPL/2.0/.

## Step 1 - Creating the [HD-BET](https://github.com/MIC-DKFZ/HD-BET) brain masks to be used in the [QUIT](https://github.com/spinicist/QUIT) MPM pipeline
- First, create and run fsl_roi.sh as documented in “Step 1” in the notebook.
- Create an SGE job file that receives the index file and and index, and run it as following:
      
      qsub -t 1:[N] brain_mask_creation.job
    where N is the number of rows in the brain_mask_creation.index file.
- **Then, run the [QUIT MPM pipeline](https://github.com/spinicist/QUIT/blob/master/Python/qipype/workflows/mpm.py) while providing the correct brain mask for each acquistion.**
- Validate your steps and results.


## Step 2 - Copying all the original MTsat files to the relevant working directory
- After the QUIT MPM pipeline finishes running, copy all the resulting MTsat MPM files to the mt_delta_template_work_dir.
- Follow step number 2 in the notebook and validate your steps and results.


## Step 3 - Running a manual quality control (QC) step
- Create an "excluded" directory under each age_group specific directory.
- Move the images that are not suitable for template creation into the /excluded directory.
- Validate your steps and results.


## Step 4 - Creating the MTsat templates per age group using [ANTs](https://github.com/ANTsX/ANTs)
- Copy the ANTs antsMultivariateTemplateConstruction2.sh [script](https://github.com/ANTsX/ANTs/blob/master/Scripts/antsMultivariateTemplateConstruction2.sh) into each one of the age_group directories under mt_delta_template_work_dir
- Edit it to your liking according to your wanted SGE runtime configuration (e.g. QSUBOPTS)
- Run, for example:
 
```./antsMultivariateTemplateConstruction2.sh -b 0 -c 1 -d 3 -i 4 -g 0.2 -k 1 -n 0 -r 1 -o ${PWD}/mt_delta_template_14_and_above_output_ ${PWD}/sub*.nii.gz```
 
- Validate your steps and results.


## Step 5 - registering the remaining excluded MTsat images into the resulting age-appropriate MTsat template using [ANTs](https://github.com/ANTsX/ANTs)
- For each age group, create and run the register_mt_to_{age_group_name}_template.sh file.
- Follow step number 5 in the notebook and validate your steps and results


## Step 6 - copying the resulting transformation files for each MTsat image in the age-appropriate MTsat template to the relevant MPM folder
- Two transformation files will be copied - the SyN deformation file (*1Warp.nii.gz file) and the affine transformation file (0GenericAffine.mat).
- The MTsat images transformed into the MTsat template space are also copied.
- Follow step number 6 in the notebook and validate your steps and results.


## Step 7 - Creating the index file to transform all the MPM images (R1, R2*, PD) from their original space into the age-appropriate MTsat template space
- To create the index files, follow step number 7 in the notebook and validate your steps and results.


## Step 8 - now, we want to transform the R1, R2* and PD images from their original space into the age-appropriate MTsat template space
- Create an SGE job file that receives the index file and and index, and run it as following:
      
      qsub -t 1:[N] mpm_to_mt_template.job  
    where N is the number of rows in the mpm_to_mt_template.index file
- Follow step number 8 in the notebook and validate your steps and results.


## Step 9 - Creating the index file to transform all the MPM images (R1, R2*, PD) from the age-appropriate MTsat template space into the MNI template space
- The MNI template chosen is the [tpl-MNIPediatricAsym cohort 5](https://www.templateflow.org/browse/) 1mm template.
- Follow step number 9 in the notebook and validate your steps and results.


## Step 10 - now, we want to transform the R1, R2* and PD images from the age-appropriate MTsat template space into the MNI template space
- Create an SGE job file that receives the index file and and index, and run it as following:

      qsub -t 1:[N] mpm_in_mt_to_mni.job
- where N is the number of rows in the mpm_in_mt_to_mni.index file.
- Follow step number 10 in the notebook and validate your steps and results.


## Step 11 - merging all MTsat images in the MNI space for a 2nd round of quality control (following registration), before any statistical analysis takes place
- Create an "fsl merge" command to concat all (before any quality control) MTsat MPM files into a single 4d nii.gz file.
- Run the resulting fsl merge command and manually perform a quality control step on the resulting 4d file. Follow a similar pattern for the 4d files of the remaining modalities as well.


## Step 12 - merging all MTsat, R1, R2*, PD images that passed QC in order to perform statistical analysis using fsl randomise on each 4d file
- Create the “fsl merge” command and run it on each modality, to create the final 4d files with the images that passed the QC step.
- Validate your steps and results.


## Step 13 - Statistical analysis:

**FSL randomise:**

- Create design and matrix files (age_corr_sorted.mat and age_corr.con, assuming each 4d file contain acquisitions from the same participants and sessions in the same temporal order).
- A mask containing only white matter and gray matter was created on top of the existing WM/GM masks for the provided MNI template and theshloded.
- Run permutation tests with age as a single covariate.
```
randomise_parallel -i 4d_all_mt_sorted.nii.gz -o mt_all_age_corr_sorted_qc2 -d age_corr_sorted.mat -t age_corr.con -m wm_gm_mask.nii.gz -n 10000 -T -D
    
randomise_parallel -i 4d_all_r1_sorted.nii.gz -o r1_all_age_corr_sorted_qc2 -d age_corr_sorted.mat -t age_corr.con -m wm_gm_mask.nii.gz -n 10000 -T -D
    
randomise_parallel -i 4d_all_r2s_sorted.nii.gz -o r2s_all_age_corr_sorted_qc2 -d age_corr_sorted.mat -t age_corr.con -m wm_gm_mask.nii.gz -n 10000 -T -D
```
- Threshold the resulting images with values above 0.95 for statistically significancant clusters.

**ROI analysis:**

- Create a binary mask for each region. For instance, in the Harvard-Oxford Subcortical Atlas, the left pallidum has a value of "7". For example:

      fslmaths HarvardOxford-sub-maxprob-thr0-1mm.nii.gz -thr 7 -uthr 7 -bin l_pallidum.nii.gz  
- For each permutation of (modality, ROI) run fslstats in order output the mean value. For example:

      fslstats -t 4d_all_mt_sorted.nii.gz -k l_pallidum.nii.gz -m
- Finally, analyse the relationship between the mean MTsat, R1 and R2* values in each ROI and age by using the output of fslstats. Our statistical analysis was performed using [Graphpad Prism](http://graphpad.com).

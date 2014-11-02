# README
---

## EndomembraneQuantifier

## Detailed Description

### 1. Overview
We developed an automated bioimage analysis algorithm called EndomembraneQuantifier (EnQu), which can subtract background signal deriving from chloroplasts, deeper plant tissues and guard cells as well as detect genuine endomembrane compartment signals with high accuracy (Beck & Zhou et al., 2012). 

This script improves extensively from the endomembrane script (Salomon et al., 2010). The new features in EnQu include quantifying colocalisation of two-channel signals (GFP and chlorophyll autofluorescence), recording the imaging time, and analysing metadata of high content image datasets. EnQu were used to monitor flg22-induced FLS2 endocytosis over time, for example FLS2-GFP endosomes appeared at 30-45 minutes after applying flg22 and the endosome number increased significantly until reaching the peak at 60-75 minutes.

### 2. The algorithm 
In order to carry out high-throughput bioimage analysis on images acquired by TSL’s (The Sainsbury Laboratory, Norwich Research Park) high content screening system Opera, we implemented EnQu on the basis of PerkinElmer's Acapella bioimage analysis platform. Endosomes were initially detected as described previously (Salomon et al., 2010) and filtered according to their roundness, intensity, area, length and width. Based on global dynamic thresholding, the remaining endosomal objects were categorised into two groups (dark region and bright region). After that, contrast values of the remaining endosomes (difference between the average intensity of a spot object and its surrounding area) were computed, based on which genuine spots from two regional groups were retained. Real endosomes in the bright-region group shall have considerably high contrast values, whereas endosomes in the dark-region group shall have relatively low contrast values. By combining the results from two groups, we could separate genuine endosomal signals from noise and imaging artefacts effectively. The bioimage analysis procedure of EnQuproduces 25 output fields. Some popularly used fields are spot number per image, valid image area on pseudo-projection images, Opera imaging time, average intensity of the recognized spots, and total number of spots in a well. 

For the detection of endomembrane vesicles labelled with two different colours (GFP and RFP) as well as to perform their colocalisation study, we created a two-channel spots-overlapping detection function and embedded it in the algorithm called EndomembraneCoLocQuantifier (EnCo). This algorithm consists of three parts: filtering background autofluorescence signals, selecting genuine endosomes from two channels, and overlapping these two-channel endosomal objects to perform a colocalisation study. The bioimage analysis procedure for EnCo produces 40 output fields, including valid image area, Opera imaging time, number of spots in a stack, overlapping spots per stack, and total number of overlapping spots per well. 

Our algorithms (in .script format) can be freely obtained from http://sourceforge.net/projects/robatzekimages/files/ as well as through our *Plant Cell* publication (http://www.plantcell.org/content/24/10/4205.full).

### 3. Software Implementation 
**Version**: 	current stable work versions: 
  * For old Opera Operating System (OS below version 2.0), V3.9
  * For recently upgraded Opera OS (version 2.0 onwards), V4.0

**Authors**: 	Dr Ji Zhou and Dr Dan MacLean, The Sainsbury Laboratory, Norwich Research Park, Norwich, UK (NR4 7UH)

**System Requirements**: To allow the use of the scripts, both Acapella software platform (version 2.0 onwards) and Windows XP are required. The scripts can also be executed on Acapella v2.6 together with Windows 7 OS.  

**Sample datasets & Acapella coding**: See tutorial.md

**Contact details**: ji.zhou@tgac.ac.uk; dan.maclean@tsl.ac.uk  

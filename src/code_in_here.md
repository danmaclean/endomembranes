Quote break.

```Acapella
//*****************************************************************************************************************
//
// File: One_Channel_Spots_Detection_JZ.script
// Version: 3.9 - added csv output files for image analysis, refined the valid image area detection
//			Improved the spots detection/filtering, and developed function to detect dark images 
//			The Sainsbury Laboratory
//			Ji Zhou
//			email: ji.zhou@tsl.ac.uk
//
//			To allow users to check the sum of small spots in 5 stacks captured by one camera only
//
// 	To allow use of this script in Acapella. This script will be transferred into a procedure which will 
// 	be placed in the standard library directory of the Opera Software
//	C:\PerkinElmerCTG\IMacro\StdLib\RobatzekProcs
//
//	DESCRIPTION:
// 	This script is produced for researchers working on analysing one channel images
//  As only one camera has been used to capture images, therefore, the sequence of images is linear
//  in the flex file - e.g., image1 to image 21 shall be merged as one max projection image
// 
//  Copyright:   (C) The Sainsbury Laboratory
//
//*****************************************************************************************************************

//set up the user input form....
input(StackNo, 0, "Stack No", "i", "Number of the stacks to analyze. If set to 0 all stacks are evaluated.")
input(CameraNo, 1, "Camera No", "i", "How many cameras have been used in the experiment")
input(ValidPercent, 65, "Valid Image Area", "i", "Specify percentage points of the valid image area - e.g. enter 50 means that if over 65% of the image is too dark or does not contain information, this image will be discarded from analysis")
input(ZplanesInStack, 21,"Zplanes in stack (per channel) ","i","Number of z-planes in stack (for each channel), a number of time moments in kinetic measurement")
input(IN_startPlane, 1,"StartPlane for Projection","i","Plane counting starts with the value 1. The specified Start Plane will be included in the projection.")
input(IN_endPlane, 21,"EndPlane for Projection","i","Plane counting starts with the value 21. The specified End Plane will be included in the projection.")
input(ShowIllustrations, YES, "Show Illustrations", "y", "YES- Output illustrations are depicted. No- Output illustrations are not shown.")
input(debugImages,NO, "Debug Using Images", "y", "YES- all intermediary images are shown.")
input(MinStdDev,4,"MinStdDev","i","Minimum standard deviation of pixel intensity in image.")
//end user input

//hardcode the number of channels for testing
set(NumberOfChannels = CameraNo)
Create("vector") 

//use singlewell to turn flex files and turn into internal data objects..
SingleWell(compact=yes)	//only the SourceData, SourceDataProp and WellIndex will be output
set( imagefilename1 = SourceData.sourcefilename[0])
RobatzekProcs::ControlImageFieldsStackAC2(sourcedata, NumberOfChannels, ZplanesInStack, StackNo) // control images' validity 

set(InvalidStacks = 0)  //counts black/grey stacks without information
set(BadStack = 0)   //counts stacks with Signal but no recognized cells
set(Img_Percent = 0)   //counts stacks with Signal but no recognized cells
set(all_VarsOK1 = 0)
set(start_time = 0)
set(end_time = 0)
set(duration = 0)
set(ALL_Duration = 0)
set(OP_NumberOfSpots_CH1 = 0)

create("objectlist")
rename(all_objects = objectlist) //one way to create an empty objectlist, same can be used for vector

Alias(_StackCounter, _FieldCounter)
Alias(StartField, StartStack)
Alias(EndField, EndStack)

// Prepare output csv file
set( namelength = length( imagefilename1 ) )
set( pathname = substr( imagefilename1, 1, namelength - 14 ) )
printfopen(pathname & "results_"& WellIndex &".csv")
Printf("Well_Index#")
Printf("Stack_No#")
Printf("Well_Location#")
Printf("Treatment#")
Printf("Opera_Start_Time#")
Printf("Opera_End_Time#")
Printf("Valid_IMG_Area#")
Printf("Detected_CH1_Spots#")
Printf("Calulated_CH1_Spots_100_IMG#\n") 
printfopen() 
// finished with the output file

Foreach(StartStack..EndStack, _StackCounter) 	
	set(BadStack = 0)  	
	Printf("Reading Stack %s \n", _StackCounter)	
	// Assigns image names for the current field, IM_CH1 - first channel image, IM_CH2 - second channel image etc
	OperaTemplates::AssignStacks()
	
	Foreach(1..NumberOfChannels,channel)
		Set(table = _["images_CH"&channel]) // _[] find container
		TableFilter(plane >= IN_startPlane and plane <= IN_endPlane) // the way to filter planes!!!
		Set(_["images_CH"&channel] = table) //assign back to images channel
	End()
	OperaTemplates::MaxProjections()
	
	Delete(start_timestamp)
	Delete(end_timestamp)
	set(start_timestamp = Images_CH1.DateTime[IN_startPlane-1])
	if((EndStack-StartStack) == 0)
		set(Start_OperaTime = start_timestamp)
	End()
	if(_StackCounter == 1 && (EndStack-StartStack) > 0)
		set(Start_OperaTime = start_timestamp)
	End()

	set(end_timestamp = Images_CH1.DateTime[IN_endPlane-1])
	set(End_OperaTime = end_timestamp)
	RobatzekProcs::Calc_Opera_time(start_timestamp, end_timestamp)	
	Set(Experiment_date = Experiment_Date)
	set(duration = total_seconds)

	//*** Detecting the valid image	area - not very precise, waiting for Achim's advice 	
	RobatzekProcs::Check_Area(IM_Max_CH1, IM_Plane_CH1)
	Set(CH1_Area_Mask = Area_Mask)
	Set(Img_percent = valid_area_percentage)	
	//*** Finish detecting valid image area ***//

	If(Img_percent < ValidPercent/100 || IM_Max_CH1.mean < 32.5) 
		// discard if the valid image area is below 50% - depends on the input threshold
		If(Img_percent < ValidPercent/100) 
			set(BadStack = 1)
			Warning("Only " & int(Img_percent * 100) & "% of the image can be analysed - image discarded.")
		End()
		// discard a very dark image 	
		If(Img_percent >= ValidPercent/100 && IM_Max_CH1.mean < 32.5) 
			set(BadStack = 1)
			Warning("The image is not valid for analysis - image discarded.")
		End()
	Else()
		// Detects the initial set of spots in channel 1
		spot_detection_c_inner(IM_Max_CH1, SpotMinimumDistance=2, SpotPeakRadius=1, SpotReferenceRadius=8, SpotMinimumContrast=0.1, SpotMinimumToCellIntensity=0.75)
		delete(spotcandidates,wholecells)		
		// Detects the actual spot locations and attributes like area, contrast, width, length etc and classifies the spots and filters the spots.
	
		// Output parameters, SpotsFiltered - objectlist 
		Delete(SpotsFiltered)
		set(IM_projected1 = IM_Max_CH1) 
		RobatzekProcs::SpotClassificationLeafCels()
	
		// Output sequence starts
		push(EntireNumberOfSpotsInStackVector, SpotsFiltered.count)
		push(SpotsFilteredVector, SpotsFiltered)
		push(imageVector, IM_Projected1)
		push(StacksUsed, _StackCounter )

		Delete(SpotsFiltered_CH1)
		ObjectFilter(area<40, objects = SpotsFiltered) // pi * 3.5 * 3.5 = 38.48 (diameter normally less than 7 pixels)
		ObjectFilter(contrast > 0.095)
		CalcAttr(Intensity_Difference_ref, ((PeakIntensity - Intensity)/ReferenceIntensity)) 
		CalcAttr(Intensity_Difference, ((PeakIntensity - ReferenceIntensity) + (PeakIntensity - Intensity))/2*contrast)  
		// in the future shall be divided by contrast
		Set(SpotsFiltered = objects)

		RemoveBorderObjects(1, objects=SpotsFiltered)
		ObjectFilter(Intensity_Difference_ref >= 0.25) // Get rid of dust 
		ObjectFilter(RoundnessCorrected > 0.5) //Find round spots
		ObjectFilter(width2lengthRatio > 0.35) //Find spots with regular shapes  
		ObjectFilter(fullLength <= 10 && ((peakintensity - intensity) >= 5))
		// filtering with contrast value and the difference with the region
		
		ObjectFilter(Intensity_Difference > 2.5)
		set(SpotsFiltered_CH1=objects) //initial refinement
		// Finish detecting spots in channel 1

		//*** Start to remove spots on cell membrane ***//
		// get rid of miss detected spots on the cell walls  	
		if(IM_projected1.fmedian <= 25 && IM_Max_CH1.max <= 125) // a very dim image
	
			if (SpotsFiltered_CH1.@count >= 5) 
			// FOR DARK IMAGES IT IS UNLIKELY THAT CELL MEMBRANE AND STOMATA ARE CLEARLY IMAGED
			
			// increasing to 5 spots in case some endosomes have been mistakenly detected
				//*** Start to remove redundant spots in a dim image ***//
				Printf("The intensity of Stack "&_StackCounter&" is very low! \n")
			
				// remove 90% of darkest pixels in remapping and brighten the image
				Expand(90, 0, image=IM_projected1)
				Set(expand_dark_image = image)
		
				// create a mask for the remapping refined_image_dark
				ThresholdXX(0.5, Image=expand_dark_image, Sqq=2.5)
				Mask(threshold = threshold*1.5, Image = expand_dark_image)
				Mask2Stencil()
				// create an object list based on the mask
				Stencil2Objects()	
				RemoveSmallObjects(10)		
				Set(ch1_membrane_dark = objects) 			
				// finishing creating dark mask objects 
	
				set(stencil = ch1_membrane_dark.body)
				// increase the border by 1 pixel, so that the fuzzy edge of the mask can be removed
				CalcErosion(1, objects = ch1_membrane_dark, Stencil = stencil) 	
				RenameAttr(body = stencil_eroded)
				Set(stencil = objects.body)
				Set(ch1_membrane_dark = objects)			
				CalcErosion(-2, objects = ch1_membrane_dark, Stencil = stencil) 	
				RenameAttr(body = stencil_eroded)
				Set(ch1_membrane_dark = objects)
		
				// calculate areas of objects in ch1_refined_image
				CalcArea()	
				ObjectFilter(area >= 15) // Get ride of small image sections (e.g., noisy signals)
				ObjectFilter(area <= 10000)
				Set(ch1_membrane_dark = objects)
	
				// overlap spot centres with the mask
				// if a spot centre does not overlap with the mask body, calcStat will return ZERO
				// otherwise a value will be returned
				CalcStat("mean", Stencil = SpotsFiltered_CH1.SpotCenters, Image = ch1_membrane_dark.body.image, AttrName="overlapped", objects = SpotsFiltered_CH1)
				// results of the calculation will be stored in a new attribute called "overlapped" in objects 
				Set(All_CH1_Spots = objects)
							
				ObjectFilter(overlapped==0, objects=All_CH1_Spots)
				Set(SpotsFiltered_CH1_membrane = objects)
				ObjectFilter((Intensity_Difference > 65 && Intensity_Difference_ref > 1.45) || (Intensity_Difference <= 65 && Intensity_Difference > 25 && Intensity_Difference_ref > 1) || ((Intensity_Difference <= 25 && Intensity_Difference >=17.5) && Intensity_Difference_ref > 0.75) || ((Intensity_Difference < 17.5 && Intensity_Difference >=10) && Intensity_Difference_ref > 0.625) || ((Intensity_Difference < 10 && Intensity_Difference >=5)  && Intensity_Difference_ref > 0.6) || (Intensity_Difference < 5 && Intensity_Difference_ref > 0.575), objects=SpotsFiltered_CH1_membrane)
				Set(SpotsFiltered_CH1_membrane = objects)
				
				ObjectFilter(overlapped > 0, objects=All_CH1_Spots)
				Set(SpotsFiltered_CH1_cell = objects)
				ObjectFilter((PeakIntensity > IM_projected1.mean * 2.15) || ((Intensity_Difference > 25 && Intensity_Difference_ref > 0.75) || ((Intensity_Difference <= 25 && Intensity_Difference >=10) && Intensity_Difference_ref > 0.6) || (Intensity_Difference < 10 && Intensity_Difference_ref > 0.465)), objects=SpotsFiltered_CH1_cell)
				Set(SpotsFiltered_CH1_cell = objects) 	
				
				//ObjectFilter(Intensity_Difference_ref > 0.5, objects=SpotsFiltered_CH1_refined) // this will get rid of many blur and dusk
				AddObjects(SpotsFiltered_CH1_membrane, objects=SpotsFiltered_CH1_cell, CheckOverlap=no)
				set(SpotsFiltered_CH1_final = objects)
				//*** end of removing redundant spots from Channel 2 ***//
			Else()
				Warning("Stack "&_StackCounter&" is too dark and does not contain enough spots - please check image " & substr(imagefilename1, length( imagefilename1 ) - 13, length( imagefilename1 )) &"!")	
				set(BadStack = 1) 
			End()

		Else()	
			// remove 75% of darkest pixels in remapping and increase the intensity of brightest pixels
			Expand(75, 0.1, image=IM_projected1) // main choroplast/stomata will be genearted from the skeleton detection 
			Set(IM_expanded = image)
			Expand(70, 0, image=IM_projected1) // main cell membrane will be generated from bright mask
			Bright_Mask(image)			
			
			// detecting cell membrane based on intensity
			ThresholdXX(1.25, Image=IM_expanded)
			Mask(threshold = threshold * 1.35, image=IM_expanded)
			//Create the initial objects
			Mask2Stencil(mask, "Joint clusters", Neighbourhood=8)
			Stencil2Objects()
			RemoveSmallObjects(25)
			CalcSkeletonByIntensity(Image=IM_expanded, SkeletonType="sideconnected", IntensityEvaluationMode=1, Objects=objects)
			set(cal_skeleton_objects = objects)
			set(mask_from_skeleton = cal_skeleton_objects.body.mask.image)

			// detecting cell membrane based on bright mask
			Mask2Stencil(M_bright, Neighbourhood=4)
			Stencil2Objects()
			RemoveSmallObjects(25)
			set(cal_BrightMask_objects = objects)
			set(mask_from_BM = cal_BrightMask_objects.body.mask.image)

			OR(mask_from_skeleton, image=mask_from_BM)
			Set(refined_membrane = image)

			mask(1, image = refined_membrane)
			Mask2Stencil(mask, "Joint clusters", Neighbourhood=8)
			Stencil2Objects()
			CalcArea(AutoRecalc=yes)
			ObjectFilter(area > 25)
			CalcErosion(-1, objects = objects, Stencil = stencil) // increasing 1 pixel
			RenameAttr(body = stencil_eroded)	
			set(stencil = objects.body)

			// increase the border by 1 pixel, so that the fuzzy edge of the mask can be removed
			// Increasing the body directly by 1 pixel will not remove the fuzzy edge - interesting...
			CalcErosion(1, objects = objects, Stencil = stencil) // decreasing 1 pixel
			RenameAttr(body = stencil_eroded)
			set(stencil = objects.body)
			CalcErosion(-1, objects = objects, Stencil = stencil) // decreasing 1 pixel
			RenameAttr(body = stencil_eroded)
			Mask2Stencil(objects.body.mask)			
			Stencil2Objects()

			CalcArea()		
			ObjectFilter(area >= 50) 
			ObjectFilter(area <= 50000) // Get ride of small image sections (e.g., noisy signals)
			object_contrast_general(reference = IM_projected1, ContrastDef = "WithoutCyto")	
			ObjectFilter(contrast > 0.1)
			Set(CH1_refined_membrane = objects)			

			// overlap spot centres with the mask
			// if a spot centre does not overlap with the mask body, calcStat will return ZERO
			// otherwise a value will be returned
			CalcStat("mean", Stencil = SpotsFiltered_CH1.body, Image = CH1_refined_membrane.body.image, AttrName="overlapped", objects = SpotsFiltered_CH1)
			// results of the calculation will be stored in a new attribute called "overlapped" in objects 
			Set(All_CH1_Spots = objects)
			
			//*** Start to filter cell spots, which might contain some spots on cell wall if the cell wall is very dark ***//
			ObjectFilter(overlapped==0, objects=All_CH1_Spots)
			Set(SpotsFiltered_CH1_cell = objects)

			ObjectFilter((PeakIntensity > IM_projected1.mean * 2.15) || ((Intensity_Difference > 25 && Intensity_Difference_ref > 0.75) || ((Intensity_Difference <= 25 && Intensity_Difference >=10) && Intensity_Difference_ref > 0.6) || (Intensity_Difference < 10 && Intensity_Difference_ref > 0.465)), objects=SpotsFiltered_CH1_cell)
			Set(SpotsFiltered_CH1_cell = objects) // can change back to  Intensity_Difference < 10 && Intensity_Difference_ref > 0.5
			// In the future, another exception can be made to accept high intensity spots!!!
		
			//*** Start to filter membrane spots ***//
			ObjectFilter(overlapped>0, objects=All_CH1_Spots)
			Set(SpotsFiltered_CH1_membrane = objects)

			ObjectFilter((Intensity_Difference > 65 && Intensity_Difference_ref > 1.45) || (Intensity_Difference <= 65 && Intensity_Difference > 25 && Intensity_Difference_ref > 1) || ((Intensity_Difference <= 25 && Intensity_Difference >=17.5) && Intensity_Difference_ref > 0.75) || ((Intensity_Difference < 17.5 && Intensity_Difference >=10) && Intensity_Difference_ref > 0.625) || ((Intensity_Difference < 10 && Intensity_Difference >=5)  && Intensity_Difference_ref > 0.6) || (Intensity_Difference < 5 && Intensity_Difference_ref > 0.575), objects=SpotsFiltered_CH1_membrane)
			Set(SpotsFiltered_CH1_membrane = objects)
		
			AddObjects(SpotsFiltered_CH1_membrane, objects=SpotsFiltered_CH1_cell, CheckOverlap=no)
			set(SpotsFiltered_CH1_final = objects)
			//*** end of removing redundant spots from Channel 1 ***//
		End()
	End()

	if (InvalidStacks<=EndField-StartField && BadStack != 1) 
		// At least one field was analysed and we have an object list
		set(OP_NumberOfSpots_1 = SpotsFiltered_CH1_final.count)
		set(OP_SpotAverageIntensity_1=SpotsFiltered_CH1_final.intensity.mean)
		set(OP_SpotAverageArea_1=SpotsFiltered_CH1_final.area.mean)
		set(OP_SpotTotalSignal_1=SpotsFiltered_CH1_final.IntegratedSpotSignal.sum)
		set(OP_SpotTotalSignal_backgroundsubtracted_1=SpotsFiltered_CH1_final.IntegratedSpotSignal_backgroundsubtracted.sum)
		set(OP_SpotAverageLength_1=SpotsFiltered_CH1_final.FullLength.mean)
		set(OP_SpotAverageHalfWidth_1=SpotsFiltered_CH1_final.HalfWidth.mean)
		set(OP_SpotAverageWidth2LengthRatio_1=SpotsFiltered_CH1_final.Width2LengthRatio.mean)
		set(OP_SpotAverageWidth2LengthRatioStddev_1=SpotsFiltered_CH1_final.Width2LengthRatio.stddev)
		set(OP_SpotAverageRoundness_1=SpotsFiltered_CH1_final.RoundnessCorrected.mean)
		set(OP_SpotAverageRoundnessStddev_1=SpotsFiltered_CH1_final.RoundnessCorrected.stddev)
		set(OP_SpotAverageContrast_1=SpotsFiltered_CH1_final.Contrast.mean)
		set(OP_SpotAverageContrastStddev_1=SpotsFiltered_CH1_final.Contrast.stddev)
		set(OP_SpotAveragePeakIntensity_1=SpotsFiltered_CH1_final.PeakIntensity.mean)
		set(OP_StackCount_1 = StackCount)	
	end()
			
	if(!BadStack)

		If(ShowIllustrations)	
		//generate some images 
		imageview(SpotsFiltered_CH1.border.mask, "original ch1 spots: stack " & _StackCounter, image=IM_Max_CH1, middlecolor=rgb(0x00,0xff,0x00))	
		imageview(SpotsFiltered_CH1_final.border.mask, "refined ch1 spots: stack " & _StackCounter, image=IM_Max_CH1, middlecolor=rgb(0x00,0xff,0x00))
		End()

		// output generated images
		set( namelength = length( imagefilename1 ) )
		set( corename = substr( imagefilename1, 1, namelength - 5 ))
		// Max Projection CH1
		Gamma(2.0, image=IM_Max_CH1)
		WriteImage(imagefile=corename & "_stack_" & _StackCounter & "-1_" & Images_CH1.AreaName[0] & "_Max_CH1.png", image=Image, imageformat="png")
		// Max Projection with original spots
		Gamma(2.0, image=IM_Max_CH1)
		CarryObjects(SpotsFiltered_CH1.border, image.max)
		CarryObjects(SpotsFiltered_CH1.border, "Green")
		WriteImage(imagefile=corename & "_stack_" &  _StackCounter & "-2_" & Images_CH1.AreaName[0] & "_CH1_Original_Spots.png", image=Image, imageformat="png")
		// Max Projection with refined spots
		Gamma(2.0, image=IM_Max_CH1)
		CarryObjects(SpotsFiltered_CH1_final.border, image.max)
		CarryObjects(SpotsFiltered_CH1_final.border, "Green")
		WriteImage(imagefile=corename & "_stack_" &  _StackCounter & "-3_" & Images_CH1.AreaName[0] & "_Ch1_Refined_Spots.png", image=Image, imageformat="png")
		
		// output in the final results
		output(Experiment_Date, "Date of the Experiment of stake " & _StackCounter & "(Ch 1)")	
		output("True", "Valid Image" & _StackCounter & "(Ch 1)")		
		output(OP_NumberOfSpots_1, "Refined Number of Spots in stack " & _StackCounter & "(Ch 1)")	
		set(OP_NumberOfSpots_CH1 = OP_NumberOfSpots_1 + OP_NumberOfSpots_CH1)	
		// output the original spots detection
		output(SpotsFiltered_CH1.@count, "Original Number of Spots in stack " & _StackCounter & "(Ch 1)")	
		// output the image analysis area
		output(int(Img_Percent * 100) & "%", "Valid image area in stack " & _StackCounter & "(Ch 1) %")	
		// output the image analysis area
		output(duration, "Seconds Opera spent picturing stack " & _StackCounter & "(Ch 1)")	

		// finish outputing final resutls   
		output("########", "----- Line Break in stack " & _StackCounter&" -----") 

		// Start to build up the output table
		printfopen(pathname & "results_"& WellIndex &".csv", yes)
		Printf(WellIndex & "#")
		Printf( _StackCounter & "#")
		Printf("R" & Images_CH1.Row[0] & " C"& Images_CH1.Column[0] & "#")
		Printf(Images_CH1.AreaName[0] & "#")
		Printf( Start_OperaTime & "#")
		Printf( End_OperaTime & "#")
		Printf( int(Img_percent * 100) &"#") 
		Printf( SpotsFiltered_CH1_final.@count & "#")
		Printf( SpotsFiltered_CH1_final.@count/Img_Percent & "#\n")
		printfopen() // finished with the output file

		// create spotsfiltered objectlist
		if(all_VarsOK1 == 0)
			set(all_SpotsFiltered_1=SpotsFiltered_CH1_final)
			set(all_VarsOK1 = all_VarsOK1 + 1)
		else()
			AddObjects(SpotsFiltered_CH1_final, objects=all_SpotsFiltered_1, CheckOverlap=no) 
			set(all_SpotsFiltered_1=objects)
		end()
		
	Else()
		// output generated images
		set( namelength = length( imagefilename1 ) )
		set( corename = substr( imagefilename1, 1, namelength - 5 ) )
		// Max Projection CH1
		Gamma(2.0, image=IM_Max_CH1)
		WriteImage(imagefile=corename & "_stack_" & _StackCounter & "-1_" & Images_CH1.AreaName[0] & "_Max_CH1.png", image=Image, imageformat="png")

		output(Experiment_Date, "Date of the Experiment of stake " & _StackCounter & "(Ch 1)")	
		// output in the final results for bad stacks
		output("False", "Valid Image" & _StackCounter & "(Ch 1)")		
		output(NAN, "Refined Number of Spots in stack " & _StackCounter & "(Ch 1)")	
		// output the original spots detection
		output(NAN, "Original Number of Spots in stack " & _StackCounter & "(Ch 1)")	
		// output the image analysis area
		output(int(Img_Percent * 100) & "%", "Valid image area in stack " & _StackCounter & "(Ch 1) %")	
		// output the image analysis area
		output(duration, "Seconds Opera spent picturing stack " & _StackCounter & "(Ch 1)")	

		// finish outputing final resutls   
		output("########", "----- Line Break in stack " & _StackCounter&" -----") 

		// Start to build up the output table
		printfopen(pathname & "results_"& WellIndex &".csv", yes)
		Printf(WellIndex & "#")
		Printf( _StackCounter & "#")
		Printf("R" & Images_CH1.Row[0] & " C"& Images_CH1.Column[0] & "#")
		Printf(Images_CH1.AreaName[0] & "#")
		Printf( Start_OperaTime & "#")
		Printf( End_OperaTime & "#")
		Printf( int(Img_percent * 100) &"#") 
		Printf( nan & "#")
		Printf( nan & "#\n")
		printfopen() // finished with the output file

	End()
End()// end of loop over stacks

If(all_VarsOK1 > 0)
// Finalise output results
	set(OP_SpotAverageIntensity_1=all_SpotsFiltered_1.intensity.mean)
	set(OP_SpotAverageArea_1=all_SpotsFiltered_1.area.mean)
	set(OP_SpotTotalSignal_1=all_SpotsFiltered_1.IntegratedSpotSignal.sum)
	set(OP_SpotTotalSignal_backgroundsubtracted_1=all_SpotsFiltered_1.IntegratedSpotSignal_backgroundsubtracted.sum)
	set(OP_SpotAverageLength_1=all_SpotsFiltered_1.FullLength.mean)
	set(OP_SpotAverageHalfWidth_1=all_SpotsFiltered_1.HalfWidth.mean)
	set(OP_SpotAverageWidth2LengthRatio_1=all_SpotsFiltered_1.Width2LengthRatio.mean)
	set(OP_SpotAverageWidth2LengthRatioStddev_1=all_SpotsFiltered_1.Width2LengthRatio.stddev)
	set(OP_SpotAverageRoundness_1=all_SpotsFiltered_1.RoundnessCorrected.mean)
	set(OP_SpotAverageRoundnessStddev_1=all_SpotsFiltered_1.RoundnessCorrected.stddev)
	set(OP_SpotAverageContrast_1=all_SpotsFiltered_1.Contrast.mean)
	set(OP_SpotAverageContrastStddev_1=all_SpotsFiltered_1.Contrast.stddev)
	set(OP_SpotAveragePeakIntensity_1=all_SpotsFiltered_1.PeakIntensity.mean)
	set(OP_StackCount_1 = StackCount)
Else()
	set(OP_SpotAverageIntensity_1=NAN)
	set(OP_SpotAverageArea_1=NAN)
	set(OP_SpotTotalSignal_1=NAN)
	set(OP_SpotTotalSignal_backgroundsubtracted_1=NAN)
	set(OP_SpotAverageLength_1=NAN)
	set(OP_SpotAverageHalfWidth_1=NAN)
	set(OP_SpotAverageWidth2LengthRatio_1=NAN)
	set(OP_SpotAverageWidth2LengthRatioStddev_1=NAN)
	set(OP_SpotAverageRoundness_1=NAN)
	set(OP_SpotAverageRoundnessStddev_1=NAN)
	set(OP_SpotAverageContrast_1=NAN)
	set(OP_SpotAverageContrastStddev_1=NAN)
	set(OP_SpotAveragePeakIntensity_1=NAN)
	set(OP_StackCount_1 = StackCount)
End()

output(OP_SpotAverageIntensity_1, "Average Intensity of Spots (Ch 1)")
output(OP_SpotAverageArea_1, "Average Area of Spots (Ch 1)")
output(OP_SpotAverageLength_1, "Average Length of Spots (Ch 1)")
output(OP_SpotAverageHalfWidth_1, "Average Half Width of Spots (Ch 1)")
output(OP_SpotAverageWidth2LengthRatio_1, "Average Width to Length Ratio of Spots (Ch 1)")
output(OP_SpotAverageWidth2LengthRatioStddev_1, "Average Width to Length Ratio of Spots - Standard Deviation (Ch 1)")
output(OP_SpotAverageRoundness_1, "Average Roundness of Spots (Ch 1)")
output(OP_SpotAverageRoundnessStddev_1, "Average Roundness of Spots - Standard Deviation (Ch 1)")
output(OP_SpotAverageContrast_1, "Average Contrast of Spots (Ch 1)")
output(OP_SpotAverageContrastStddev_1, "Average Contrast of Spots - Standard Deviation (Ch 1)")
output(OP_SpotAveragePeakIntensity_1, "Average Peak Intensity of Spots (Ch 1)")

output("########", "----- Overall Results -----") 
output(OP_StackCount_1, "Total number of Stacks analyzed in Well (Ch 1)")
output(OP_NumberOfSpots_CH1, "Number of Spots in whole Well (Ch 1)")
// Calculate Opera Imaging Period
RobatzekProcs::Calc_Opera_time(Start_OperaTime, End_OperaTime)	
set(ALL_Duration = total_seconds)
// Continue output results
output(ALL_Duration, "Total seconds of Opera's analysis in Well: ")
output(Start_OperaTime, "Opera started analysing Well at: ")
output(End_OperaTime, "Opera finished analysing Well at: ")

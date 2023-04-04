Test Image Set Processing

Description:
This program does either temporal analysis or spatial analysis on a set of images. Temporal analysis determines the percent variation from pulse
to pulse, normalized for exposure time. Spatial analysis determines the spatial distribution of the illumination and how reliable that
distribution is. In both cases the images are reduced from their original resolution by removing 25% of the image on all four sides. Once analysis is
finished the resultant data is exported to an Excel file and displayed as an image.
The two types of analysis seek to answer the following questions:

Temporal:
 	- How much percent variation do you see from pulse to pulse (as represented by Coefficient of Variation (CoV))?
	- Is it random noise or does it drift in one direction over time? why? (Is the change in CoV the same every time for the different exposures?)
	- Are the detected pulses linear with time? (will twice the exposure get you twice as much light?) If not, why? (flattened light dose might mean 
	  saturated image)
	- How much do you lose in percent compared to the 1 ms case?

Spatial:
 	- What does the spatial distribution of the illumination look like?
	- How repeateable/reliable is that illumination shape? (for a given exposure time, can you show me a picture representing the (pixel-wise) 
	  variation you see from image to image, compared to an average image?)
	- How do these factors change as you change the pulse length? (Does the illumination shape change at higher exposures?)
	BONUS: Why is it brighter in the center? Can you make it flatter?


Dependencies:
- numpy
- opencv-python
- glob2
- xlsxwriter
- matplotlib

Usage:
Global Variables
	- TEST_CASE		Determines whether the program does temporal or spatial analysis. Setting TEST_CASE to True causes the program
				run temporal analysis. Setting TEST_CASE to False causes the program to run spatial analysis
	- TEMPORAL_FILEPATH	A partial filepath for where the images for temporal analysis are stored, as well as where the Excel file with
				the analysis results is created
	- SPATIAL_FILEPATH	A partial filepath for where the images for spatial analysis are stored, as well as where the Excel file with
				the analysis results is created
	- RESOLUTION_HEIGHT	Vertical pixel distance of images to be analyzed
	- RESOLUTION_WIDTH	Horizontal pixel distance of images to be analyzed

Notes on Global Variables
- TEST_CASE should be set to True for temporal analysis and False for spatial analysis
- TEMPORAL_FILEPATH and SPATIAL_FILEPATH will need to be rewritten for use on any other PC. Each should point to the folders where their images are stored (without the ending \ that it would normally have)

Notes on Enum class LED_color
- This enumerator class represents the light conditions for the four images to be captured for temporal analysis. They are red, green, blue, and ambient light.
- This class can be added to in the future to include other forms of light, like white light from the LED. The ambient light case should always be last so that the ordered_import_images() function separates it from the others and the normalize_for_exposure() function subtract its pixel values from each of the other images' pixel values.

Notes on Dictionary exposure_time
- This dictionary correlates the LED colors used in the LED_color enumerator class with their corresponding exposure times.
- It should always have a number of entries equal to one less than the number of entries in the LED_color class. Similarly to the LED_color class it can be added to later to include more exposure times if more images are acquired. Care should be taken to ensure that the exposure times and LED colors (or their assigned names) match up.

Notes on functions
ordered_import_images()
- In order for this function to import the images in the correct sequence the images must be saved with the correct name formatting. Each image's name should have LED_color enumerator class name (RED, GREEN, BLUE, and NO) as the end text in the image file name. That is, "1__8000us_exposure_aligned_depth_single_RED.png" must have the word RED in all caps at the end of the file name as seen. This means that if the analysis code is adjusted to include more test cases then the Intel RealSense camera code must be adjusted as well to save the image files with the appropriate file names. Alternatively, if the program user knows which original file names correspond to which test cases they may add the appropriate text in the file names manually.
- If the function doesn't read in a number of images that corresponds with the LED_color enumerator class then the normalize_for_exposure() function will fail due to indexing out, because the for loop is based on the length of the LED_color class.

Notes on Results
- When the analysis data is displayed as an image, dark areas correspond to areas of low CoV percentage and light areas correspond to areas of high Cov percentage
- Data exported to an Excel file can be heat-mapped for easy visualization of the data. This can be done in Excel by going to Home -> Conditional Formatting -> Color Scales -> Red - Yellow - Green Color Scale. This causes areas of high CoV to appear more red and areas of low CoV to appear more green

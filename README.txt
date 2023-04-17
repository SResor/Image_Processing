# post_processing.py
#
# Description:
# This program does either temporal analysis or spatial analysis on a set of images. Temporal analysis determines
# the percent variation from pulse to pulse, normalized for exposure time. Spatial analysis determines the
# spatial distribution of the illumination and how reliable that distribution is. In both cases the images are
# reduced from their original resolution by removing 25% of the image on all four sides. Once analysis is finished
# the resultant data is exported to an Excel file and displayed as an image.
# The two types of analysis seek to answer the following questions:
#
# Temporal:
#  	- How much percent variation do you see from pulse to pulse (as represented by Coefficient of Variation (CoV))?
# 	- Is it random noise or does it drift in one direction over time? why? (Is the change in CoV the same
# 	  every time for the different exposures?)
# 	- Are the detected pulses linear with time? (will twice the exposure get you twice as much light?) If
#  	  not, why? (flattened light dose might mean saturated image)
# 	- How much do you lose in percent compared to the 1 ms case?
#
# Spatial:
#  	- What does the spatial distribution of the illumination look like?
# 	- How repeateable/reliable is that illumination shape? (for a given exposure time, can you show me
# 	  a picture representing the (pixel-wise) variation you see from image to image, compared to an average image?)
# 	- How do these factors change as you change the pulse length? (Does the illumination shape change at
# 	  higher exposures?)
# 	BONUS: Why is it brighter in the center? Can you make it flatter?
#
# Python Packages:
# - numpy
# - opencv-python
# - glob2
# - xlsxwriter
# - matplotlib

import cv2
import numpy as np
import glob
from enum import Enum
import xlsxwriter as xw
import matplotlib.pyplot as plt

TEST_CASE = False    # Determine whether the program runs the temporal or spatial processing code
                    # True is for Temporal, False is for Spatial

# Temporal test case partial filepath. Must be modified for use on other PCs. Should point to location of test images
# Filepath is partial because images must be loaded in a particular order
TEMPORAL_FILEPATH = r"C:\Users\seanr\PycharmProjects\Image_Processing\Temporal Test Images"

# Spatial test case filepath. Must be modified for use on other PCs. Should point to location of test images
SPATIAL_FILEPATH = r"C:\Users\seanr\PycharmProjects\Image_Processing\Spatial Test Images"

RESOLUTION_HEIGHT = 480  # Pixel height of the image files being processed
RESOLUTION_WIDTH = 640  # Pixel width of the image files being processed


# Enum class for the four different LED conditions for the temporal test case
# May need to be modified later if test case includes white light from LED
# If modified then the Intel RealSense camera code should also be modified
class LED_color(Enum):
    RED = 0
    GREEN = 1
    BLUE = 2
    NO = 3


# Dictionary to hold exposure times (in ms) with the respective LED colors for the temporal test case
# Must be modified if other exposure times are used for testing
# Length of this dictionary should always be one less than the length of the LED_color enumerator class
exposure_time = {
    LED_color.RED: 1.0,
    LED_color.GREEN: 2.0,
    LED_color.BLUE: 3.0
}


# Function to import image files in a particular order for the temporal test case
# Because the pixel values must be divided by a different exposure time for each image the images must be in a
# particular order
def ordered_import_images(filepath):
    reference_images = []
    for color in LED_color:
        end_of_file = "\\*" + str(color.name) + ".png"
        single_image = [cv2.imread(file, cv2.IMREAD_GRAYSCALE) for file in glob.glob(filepath + end_of_file)]

        if color.value == 3:
            ambient_light_image = single_image
        else:
            reference_images.append(single_image)

    print("Images imported...")

    return ambient_light_image, reference_images


# Function to import image files in no particular order for the spatial test case
def import_images(filepath):
    end_of_file = "\\*.png"
    reference_images = [cv2.imread(file, cv2.IMREAD_GRAYSCALE) for file in glob.glob(filepath + end_of_file)]

    print("Images imported...")

    return reference_images


# Function that subtracts the ambient light pixel values from each image and then normalizes the image pixel values
# for the differing exposure times by dividing out the exposure time
def normalize_for_exposure(ambient_light_image, reference_images):
    normalized_test_images = []
    single_image = np.empty([int(RESOLUTION_HEIGHT * 0.5), int(RESOLUTION_WIDTH * 0.5)])
    for color in LED_color:
        if color.value == len(LED_color) - 1:
            continue
        else:
            x = int(RESOLUTION_HEIGHT * 0.25)
            j = 0
            while x < RESOLUTION_HEIGHT * 0.75:
                y = int(RESOLUTION_WIDTH * 0.25)
                k = 0
                while y < RESOLUTION_WIDTH * 0.75:
                    ref_pixel = reference_images[color.value][0][x][y]
                    amb_pixel = ambient_light_image[0][x][y]
                    single_pixel = ref_pixel - amb_pixel
                    exposure = list(exposure_time.values())[color.value]
                    single_image[j][k] = single_pixel / exposure    # = (ref_pixel / exposure) - amb_pixel
                                                                    # should be used if the camera's exposure time is
                                                                    # not consistent across image acquisitons
                    y += 1
                    k += 1
                x += 1
                j += 1
            normalized_test_images.append(single_image.copy())

    return normalized_test_images


# Function was working with the spatial image files. The ref_array parameter must be in a list
# in order for the function to work properly, even if it's a list of length one. This is so
# that the function could be passed a list of multiple images to write to Excel, in case a user
# needs to investigate individual image values.
def write_to_excel(ref_array, filepath):
    workbook = xw.Workbook(filepath + '\\test.xlsx')

    for x in range(len(ref_array)):
        out_array = ref_array[x].copy()
        rotated_array = np.rot90(np.fliplr(out_array))

        if len(ref_array) > 1:
            worksheet = workbook.add_worksheet(str(list(exposure_time.values())[x]) + "ms " +
                                               str(list(exposure_time.keys())[x]))
        else:
            worksheet = workbook.add_worksheet("% CoV")

        row = 0

        for col, data in enumerate(rotated_array):
            worksheet.write_column(row, col, data)

    workbook.close()


# Function that will display the passed image in grayscale. The ref_array parameter must be in
# a list in order for the function to work properly, even if it's a list of length one. This is so
# that the function could be passed a list of multiple images to display if necessary.
# NOTE: darker areas correspond to locations of lowest variance, not highest
def show_image(ref_array):
    for x in range(len(ref_array)):
        out_array = ref_array[x].copy()

        plt.imshow(out_array, cmap='gray', vmin=0, vmax=255)
        plt.show()


# This function calculates the coefficient of variation (CoV) for each pixel
# in the set of images passed.
def coefficient_of_variation(reference_images, number_of_images):
    sum_image = np.empty([int(RESOLUTION_HEIGHT * 0.5), int(RESOLUTION_WIDTH * 0.5)])
    i = 0
    for x in reference_images:
        sum_image = sum_image + x

    mean_reference = sum_image / float(number_of_images)

    std_dev = np.empty([int(RESOLUTION_HEIGHT * 0.5), int(RESOLUTION_WIDTH * 0.5)])
    cov = np.empty([int(RESOLUTION_HEIGHT * 0.5), int(RESOLUTION_WIDTH * 0.5)])

    x = 0
    while x < RESOLUTION_HEIGHT * 0.5:
        y = 0
        while y < RESOLUTION_WIDTH * 0.5:
            my_vars = []
            z = 0
            while z < len(reference_images):
                var_name = reference_images[z][x][y]
                my_vars.append(var_name)
                z += 1

            std_dev[x][y] = np.std(my_vars)
            y += 1
        x += 1

    cov = std_dev / mean_reference * 100.0

    return cov


# This function trims the image from its original resolution to a smaller resolution.
# It cuts 25% of the image off on either side and the top and bottom.
def trim_image(reference_images):
    zoomed_images = []
    single_image = np.empty([int(RESOLUTION_HEIGHT * 0.5), int(RESOLUTION_WIDTH * 0.5)])

    x = 0
    while x < len(reference_images):

        y = int(RESOLUTION_HEIGHT * 0.25)
        j = 0
        while y < RESOLUTION_HEIGHT * 0.75:
            z = int(RESOLUTION_WIDTH * 0.25)
            k = 0
            while z < RESOLUTION_WIDTH * 0.75:
                single_image[j][k] = reference_images[x][y][z]
                z += 1
                k += 1
            y += 1
            j += 1
        x += 1
        zoomed_images.append(single_image.copy())

    return zoomed_images


def main():
    if TEST_CASE:
        ambient_light_image, reference_images = ordered_import_images(TEMPORAL_FILEPATH)
        normalized_reference = normalize_for_exposure(ambient_light_image, reference_images)

        # The two commented lines below can be uncommented if the user wishes to examine the normalized values
        #   of the images returned by normalize_for_exposure()
        # write_to_excel(normalized_reference, TEMPORAL_FILEPATH)
        # show_image(normalized_reference)

        cov = coefficient_of_variation(normalized_reference, len(normalized_reference))
        listr = [cov]           # Array must be held inside a list for the write_to_excel() and show_image() functions
        write_to_excel(listr, TEMPORAL_FILEPATH)
        show_image(listr)

    elif not TEST_CASE:
        reference_images = import_images(SPATIAL_FILEPATH)
        zoomed_images = trim_image(reference_images)
        cov = coefficient_of_variation(zoomed_images, len(zoomed_images))
        listr = [cov]           # Array must be held inside a list for the write_to_excel() and show_image() functions
        write_to_excel(listr, SPATIAL_FILEPATH)
        show_image(listr)


main()

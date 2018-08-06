# Fast (Convolution) Blur

Difficulty: 2 / 5

Slowness: scales with area of input image, unaffected by blur radius

## Overview
Fast Blur is an extremely fast image blurring algorithm, based off of algorithm 4 from http://blog.ivank.net/fastest-gaussian-blur.html (the name "fast blur" is unofficial, but is better than "gaussian" - it is important to make the distinction as they do not produce identical results)

![Imgur](https://i.imgur.com/EzaT5mJ.png)

The basic idea is that, by applying an infinite number of box blurs to an image, you will end up with a perfect Gaussian blur. However this is not practical, so instead a limited number of box blurs are applied to approximate Gaussian blur. Since these algorithms can run extremely fast, the overall effect is a high speed blurring algorithm.

## Implementation

Note: the same Bitmaps.java file is used here as in the gaussian blur algorithm.

### FastBlur.java
This file contains all of the code needed to process images. Similar to the gaussian blur algorithm, there are two options for handling outside pixels, but instead of creating and reusing blur objects, you pass the radius directly as no kernel or weights are needed.

The box blurring functions are highly optimised to run at the same speed regardless of blur radius. This contributes to the length of the code but the pay off is worth the added complexity.

```Java
package elttob.blur;

import java.awt.image.BufferedImage;

public class FastBlur {
	
	// how to handle pixels outside the image
	// ZERO: set to transparent
	// CLAMP: set to nearest pixel's colour
	public static enum EdgeFill { ZERO, CLAMP } 
	
	// how many convolutions (number of times to apply blur)
	public static final int CONVOLUTIONS = 3;
	// sigmaRatio: the ratio of sigma to blur radius (this value works pretty well)
	public static final double sigmaRatio = 9/16d;
	
	// applies fast blur to the input image, with the given radius and outside pixel handling
	public static BufferedImage fastConvolutionBlur(BufferedImage input, int radius, EdgeFill edgeFill) {
		// we repeatedly blur this and store the result back in this variable
		BufferedImage result = input;

		// yay maths!
		// All of this calculates which radii to use when applying the box blurs.
		// I'm not exactly sure how it works, but it works!
		double sigma = radius * sigmaRatio;
		
		double wIdeal = Math.sqrt((12 * sigma * sigma / CONVOLUTIONS)+1);
		double wl = Math.floor(wIdeal); if(wl % 2 == 0) wl--;
		double wu = wl + 2;
		
		double mIdeal = 
				(12 * sigma * sigma - CONVOLUTIONS * wl * wl - 4 * CONVOLUTIONS * wl - 3 * CONVOLUTIONS) 
				/ (-4 * wl - 4);
		
		double m = Math.floor(mIdeal + 0.5d);
		
		// for each convolution (each time to apply a box blur)
		for(int c = 0; c < CONVOLUTIONS; c++) {
			int thisRadius = (int) (c<m?wl:wu);
			thisRadius = (thisRadius - 1) / 2;
			// apply the box blur!
			result = fastBoxBlurY(fastBoxBlurX(result, thisRadius, edgeFill), thisRadius, edgeFill);
		}
		return result;
	}
	
	// shortcut for lazy people
	public static BufferedImage fastBoxBlur(BufferedImage input, int radius, EdgeFill edgeFill) {
		return fastBoxBlurY(fastBoxBlurX(input, radius, edgeFill), radius, edgeFill);
	}
	
	// gamma correct box blur on the x axis
	public static BufferedImage fastBoxBlurX(BufferedImage input, int radius, EdgeFill edgeFill) {
		// size of the actual box blur
		int size = radius*2 + 1;
		// the output image we'll write to later
		BufferedImage output = new BufferedImage(input.getWidth(), input.getHeight(), input.getType());
		// saving ourselves some writing
		int imgWidth = input.getWidth();
		int imgHeight = input.getHeight();
		// setting up the int arrays we'll use as bitmaps to read/write from/to
		int[] bitmapInput = Bitmaps.get(input);
		int[] bitmapOutput = new int[imgWidth * imgHeight * 4];
		
		// for each row
		for(int y = 0; y < imgHeight; y++) {
			// calculate the first part of the index for array accesses
			int rowIndex = y*imgWidth;
			
			// these are the totals for all the nearby pixels' values
			int lastr = 0, lastg = 0, lastb = 0, lasta = 0;
			// calculate totals for first pixel
			for(int i = -radius; i <= radius; i++) {
				int r = 0, g = 0, b = 0, a = 0;
				if(i < 0 || i >= imgWidth) {
					// handle pixels outside the image
					if(edgeFill == EdgeFill.ZERO) r = g = b = a = 0;
					else if(edgeFill == EdgeFill.CLAMP) {
						int local = i < 0 ? 0 : (i >= imgWidth ? imgWidth - 1 : i);
						r = bitmapInput[(rowIndex + local)*4	];
						g = bitmapInput[(rowIndex + local)*4 + 1];
						b = bitmapInput[(rowIndex + local)*4 + 2];
						a = bitmapInput[(rowIndex + local)*4 + 3];
					}
				} else {
					// handle pixels inside the image
					r = bitmapInput[(rowIndex + i)*4	];
					g = bitmapInput[(rowIndex + i)*4 + 1];
					b = bitmapInput[(rowIndex + i)*4 + 2];
					a = bitmapInput[(rowIndex + i)*4 + 3];
				}
				// add the values on to the totals
				lastr += r; lastg += g; lastb += b; lasta += a;
			}
			
			// using the totals of the nearby pixels' values, divide by the box blur size to get the mean average of the values
			bitmapOutput[rowIndex*4	] = lastr / size;
			bitmapOutput[rowIndex*4 + 1] = lastg / size;
			bitmapOutput[rowIndex*4 + 2] = lastb / size;
			bitmapOutput[rowIndex*4 + 3] = lasta / size;
			
			// now for the rest of the image
			for(int x = 1; x < imgWidth; x++) {
				
				// remove the first pixel from last time since we moved over by one
				{
					int r = 0, g = 0, b = 0, a = 0;
					// this is the actual x value we're focusing on
					int i = x - radius - 1;

					// now we get it's values
					if(i < 0 || i >= imgWidth) {
						if(edgeFill == EdgeFill.ZERO) r = g = b = a = 0;
						else if(edgeFill == EdgeFill.CLAMP) {
							int local = i < 0 ? 0 : (i >= imgWidth ? imgWidth - 1 : i);
							r = bitmapInput[(rowIndex + local)*4	];
							g = bitmapInput[(rowIndex + local)*4 + 1];
							b = bitmapInput[(rowIndex + local)*4 + 2];
							a = bitmapInput[(rowIndex + local)*4 + 3];
						}
					} else {
						r = bitmapInput[(rowIndex + i)*4	];
						g = bitmapInput[(rowIndex + i)*4 + 1];
						b = bitmapInput[(rowIndex + i)*4 + 2];
						a = bitmapInput[(rowIndex + i)*4 + 3];
					}
					// take it off the totals
					lastr -= r; lastg -= g; lastb -= b; lasta -= a;
				}
				
				// add next pixel in the row, same story here
				{
					int r = 0, g = 0, b = 0, a = 0;
					int i = x + radius;
					if(i < 0 || i >= imgWidth) {
						if(edgeFill == EdgeFill.ZERO) r = g = b = a = 0;
						else if(edgeFill == EdgeFill.CLAMP) {
							int local = i < 0 ? 0 : (i >= imgWidth ? imgWidth - 1 : i);
							r = bitmapInput[(rowIndex + local)*4	];
							g = bitmapInput[(rowIndex + local)*4 + 1];
							b = bitmapInput[(rowIndex + local)*4 + 2];
							a = bitmapInput[(rowIndex + local)*4 + 3];
						}
					} else {
						r = bitmapInput[(rowIndex + i)*4	];
						g = bitmapInput[(rowIndex + i)*4 + 1];
						b = bitmapInput[(rowIndex + i)*4 + 2];
						a = bitmapInput[(rowIndex + i)*4 + 3];
					}
					lastr += r; lastg += g; lastb += b; lasta += a;
				}
				
				// now we have the correct totals for this pixel, we divide by size to get the mean again.
				bitmapOutput[(rowIndex + x)*4	] = lastr / size;
				bitmapOutput[(rowIndex + x)*4 + 1] = lastg / size;
				bitmapOutput[(rowIndex + x)*4 + 2] = lastb / size;
				bitmapOutput[(rowIndex + x)*4 + 3] = lasta / size;
			}
		}
		
		// once we did all of that, write the final array to the output image and return it
		Bitmaps.put(output, bitmapOutput);
		return output;
	}
	
	// same as above, for the y axis. self explanatory
	public static BufferedImage fastBoxBlurY(BufferedImage input, int radius, EdgeFill edgeFill) {
		int size = radius*2 + 1;
		BufferedImage output = new BufferedImage(input.getWidth(), input.getHeight(), input.getType());
		int imgWidth = input.getWidth();
		int imgHeight = input.getHeight();
		int[] bitmapInput = Bitmaps.get(input);
		int[] bitmapOutput = new int[imgWidth * imgHeight * 4];
		
		for(int x = 0; x < imgWidth; x++) {
			
			int lastr = 0, lastg = 0, lastb = 0, lasta = 0;
			// calculate value of first pixel
			for(int i = -radius; i <= radius; i++) {
				int r = 0, g = 0, b = 0, a = 0;
				if(i < 0 || i >= imgHeight) {
					if(edgeFill == EdgeFill.ZERO) r = g = b = a = 0;
					else if(edgeFill == EdgeFill.CLAMP) {
						int local = i < 0 ? 0 : (i >= imgHeight ? imgHeight - 1 : i);
						r = bitmapInput[(x + local*imgWidth)*4	];
						g = bitmapInput[(x + local*imgWidth)*4 + 1];
						b = bitmapInput[(x + local*imgWidth)*4 + 2];
						a = bitmapInput[(x + local*imgWidth)*4 + 3];
					}
				} else {
					r = bitmapInput[(x + i*imgWidth)*4	];
					g = bitmapInput[(x + i*imgWidth)*4 + 1];
					b = bitmapInput[(x + i*imgWidth)*4 + 2];
					a = bitmapInput[(x + i*imgWidth)*4 + 3];
				}
				lastr += r; lastg += g; lastb += b; lasta += a;
			}
			
			bitmapOutput[x*4	] = lastr / size;
			bitmapOutput[x*4 + 1] = lastg / size;
			bitmapOutput[x*4 + 2] = lastb / size;
			bitmapOutput[x*4 + 3] = lasta / size;
			
			for(int y = 1; y < imgHeight; y++) {
				
				// remove first pixel
				{
					int r = 0, g = 0, b = 0, a = 0;
					int i = y - radius - 1;
					if(i < 0 || i >= imgHeight) {
						if(edgeFill == EdgeFill.ZERO) r = g = b = a = 0;
						else if(edgeFill == EdgeFill.CLAMP) {
							int local = i < 0 ? 0 : (i >= imgHeight ? imgHeight - 1 : i);
							r = bitmapInput[(x + local*imgWidth)*4	];
							g = bitmapInput[(x + local*imgWidth)*4 + 1];
							b = bitmapInput[(x + local*imgWidth)*4 + 2];
							a = bitmapInput[(x + local*imgWidth)*4 + 3];
						}
					} else {
						r = bitmapInput[(x + i*imgWidth)*4	];
						g = bitmapInput[(x + i*imgWidth)*4 + 1];
						b = bitmapInput[(x + i*imgWidth)*4 + 2];
						a = bitmapInput[(x + i*imgWidth)*4 + 3];
					}
					lastr -= r; lastg -= g; lastb -= b; lasta -= a;
				}
				
				// add next pixel
				{
					int r = 0, g = 0, b = 0, a = 0;
					int i = y + radius;
					if(i < 0 || i >= imgHeight) {
						if(edgeFill == EdgeFill.ZERO) r = g = b = a = 0;
						else if(edgeFill == EdgeFill.CLAMP) {
							int local = i < 0 ? 0 : (i >= imgHeight ? imgHeight - 1 : i);
							r = bitmapInput[(x + local*imgWidth)*4	];
							g = bitmapInput[(x + local*imgWidth)*4 + 1];
							b = bitmapInput[(x + local*imgWidth)*4 + 2];
							a = bitmapInput[(x + local*imgWidth)*4 + 3];
						}
					} else {
						r = bitmapInput[(x + i*imgWidth)*4	];
						g = bitmapInput[(x + i*imgWidth)*4 + 1];
						b = bitmapInput[(x + i*imgWidth)*4 + 2];
						a = bitmapInput[(x + i*imgWidth)*4 + 3];
					}
					lastr += r; lastg += g; lastb += b; lasta += a;
				}
				
				bitmapOutput[(x + y*imgWidth)*4	] = lastr / size;
				bitmapOutput[(x + y*imgWidth)*4 + 1] = lastg / size;
				bitmapOutput[(x + y*imgWidth)*4 + 2] = lastb / size;
				bitmapOutput[(x + y*imgWidth)*4 + 3] = lasta / size;
			}
		}
		
		Bitmaps.put(output, bitmapOutput);
		return output;
	}
	
}
```

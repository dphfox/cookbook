# Gaussian Blur

Difficulty: 2/5

Slowness: scales with area of input image and area of blur kernel

## Overview
Gaussian Blur is a simple symmetrical blurring algorithm which produces very smooth results.

![Imgur](https://i.imgur.com/FKBfqCY.png)

The basic idea is that you can calculate a blur kernel's weights using a Gaussian equation. This equation takes x and y coordinates, and a sigma value that controls the blur's 'quality'. I found that `(9/16 * radius)` works well as a sigma value. From there, it works identically to any other kernel-based blurring algorithms.

## Implementation

### Blur.java
Deals wih blur kernel generation and application. Using the static methods you can easily build a blur kernel with a given radius.
```Java
package elttob.blur;

import java.awt.image.BufferedImage;

public class Blur {
	
	// how to handle pixels outside the image
	// ZERO: set to transparent
	// CLAMP: set to nearest pixel's colour
	public static enum EdgeFill { ZERO, CLAMP }
	
	// size of the blur kernel
	int size;
	// weights in the kernel
	double[] weights;
	// sum of all weights
	double sum, sumSq;
	
	// initialise a blur kernel with a certain side length (size) and weights
	private Blur(int size, double[] weights) {
		this.size = size;
		this.weights = weights;
		
		double sum = 0;
		for(double i: weights) sum += i;
		
		this.sum = sum;
		this.sumSq = sum * sum;
	}
	
	// creates a simple (all weights = 1) blur kernel with the given radius
	public static Blur createSimpleBlur(int r) {
		if(r <= 0) return new Blur(1, new double[]{1});
		
		int size = r*2 + 1;
		double[] weights = new double[size];
		for(int i = 0; i < size; i++) weights[i] = 1;
		return new Blur(size, weights);
	}
	
	// ratio of sigma to blur radius (this value works pretty well)
	public static final double sigmaRatio = 9/16d;
	// creates a gaussian blur kernel with the given radius
	public static Blur createGaussianBlur(int r) {
		if(r <= 0) return new Blur(1, new double[]{1});
		
		// okay, I don't actually fully understand some of this math. It works tho so who cares ¯\_(ツ)_/¯
		int size = r*2 + 1;
		double[] weights = new double[size];
		double sigma = r * sigmaRatio;
		for(int x = -r; x <= r; x++)
			weights[x+r] = Math.exp(-(x*x)/(2d*sigma*sigma));
		
		return new Blur(size, weights);
	}
	
	// applies this blur kernel to the input image, using the given edge fill rule to deal with outside pixels
	public BufferedImage apply(BufferedImage input, EdgeFill edgeFill) {
		// this output image will be written to later
		BufferedImage output = new BufferedImage(input.getWidth(), input.getHeight(), input.getType());
		// saves a bunch of function calls
		int imgWidth = input.getWidth();
		int imgHeight = input.getHeight();
		// convert input image into int array for fast access
		int[] bitmapInput = Bitmaps.get(input);
		// create output int array to write to for fast writing, will end up in output bufferedimage
		int[] bitmapOutput = new int[imgWidth * imgHeight * 4];
		for(int x = 0; x < imgWidth; x++) for(int y = 0; y < imgHeight; y++) {
			
			// these variables store the total of the weighted pixel values for each channel
			double rsum = 0, gsum = 0, bsum = 0, asum = 0;
			//iterate from -radius to +radius in both x and y directions (to create a square we can use to sample all the pixels in the kernel's radius)
			for(int i = -(size-1)/2; i <= (size-1)/2; i++) for(int j = -(size-1)/2; j <= (size)/2; j++) {
				// local x and y coordinates (current x and y + x and y within the kernel square)
				int lx = x + i, ly = y + j;
				// get the rgba for this pixel
				int r = 0, g = 0, b = 0, a = 0;
				if(lx < 0 || ly < 0 || lx >= imgWidth || ly >= imgHeight) {
					// edge case (haha)
					if(edgeFill == EdgeFill.ZERO) {
						r = g = b = a = 0;
					} else if (edgeFill == EdgeFill.CLAMP) {
						//get nearest pixel
						int lx2 = lx < 0 ? 0 : (lx >= imgWidth ? imgWidth - 1 : lx);
						int ly2 = ly < 0 ? 0 : (ly >= imgHeight ? imgHeight - 1 : ly);
						
						int index = ly2 * imgWidth + lx2;
						r = bitmapInput[index*4	];
						g = bitmapInput[index*4 + 1];
						b = bitmapInput[index*4 + 2];
						a = bitmapInput[index*4 + 3];
					}
				} else {
					// just get the rgb
					int index = ly * imgWidth + lx;
					r = bitmapInput[index*4	];
					g = bitmapInput[index*4 + 1];
					b = bitmapInput[index*4 + 2];
					a = bitmapInput[index*4 + 3];
				}
				
				// calculate the weight based on our coordinates within the kernel square
				int indexX = i + (size-1)/2;
				int indexY = j + (size-1)/2;
				double weight = weights[indexX] * weights[indexY];
				
				// add to the totals with the weighted pixel values
				rsum += r * weight;
				gsum += g * weight;
				bsum += b * weight;
				asum += a * weight;
			}
			
			// divide the totals by the sum of all weights squared (normalises the totals to the 0-255 range, ideally)
			rsum /= sumSq;
			gsum /= sumSq;
			bsum /= sumSq;
			asum /= sumSq;
			
			// write to the output int array, with some sanity clamping
			int index = y * imgWidth + x;
			bitmapOutput[index*4	] = rsum < 0 ? 0 : (rsum > 255 ? 255 : (int)rsum);
			bitmapOutput[index*4 + 1] = gsum < 0 ? 0 : (gsum > 255 ? 255 : (int)gsum);
			bitmapOutput[index*4 + 2] = bsum < 0 ? 0 : (bsum > 255 ? 255 : (int)bsum);
			bitmapOutput[index*4 + 3] = asum < 0 ? 0 : (asum > 255 ? 255 : (int)asum);
			
		}
		
		// now just dump it in the output bufferedimage and hand it back
		Bitmaps.put(output, bitmapOutput);
		return output;
	}
	
}
```

### Bitmaps.java
Utility class to convert between buffered images and int arrays, since int arrays are *so much faster*. Array is laid out according to this rule: `[(y*width + x)*4 + c]` where c = 0 for red, 1 for green, 2 for blue, 3 for alpha.

```Java
package elttob.blur;

import java.awt.image.BufferedImage;

public class Bitmaps {
	
	// pretty self explanatory stuff, turns bufferedimage -> int array
	public static int[] get(BufferedImage input) {
		int[] bitmap = new int[input.getWidth() * input.getHeight() * 4];
		for(int i = 0; i < input.getWidth(); i++) for(int j = 0; j < input.getHeight(); j++) {
			int col = input.getRGB(i, j);
			int index = j * input.getWidth() + i;
			bitmap[index*4	] = ((col >> 16) & 0xFF);
			bitmap[index*4 + 1] = ((col >>  8) & 0xFF);
			bitmap[index*4 + 2] = ((col	  ) & 0xFF);
			bitmap[index*4 + 3] = ((col >> 24) & 0xFF);
		}
		return bitmap;
	}
	
	// same as above, but the other way (int array -> bufferedimage)
	public static void put(BufferedImage output, int[] bitmap) {
		for(int i = 0; i < output.getWidth(); i++) for(int j = 0; j < output.getHeight(); j++) {
			int index = j * output.getWidth() + i;
			int r = bitmap[index*4	];
			int g = bitmap[index*4 + 1];
			int b = bitmap[index*4 + 2];
			int a = bitmap[index*4 + 3];
			output.setRGB(i, j, (int)r << 16 | (int)g << 8 | (int)b | (int)a << 24);
		}
	}

}
```

(yes I use tab characters, fite me)

## Optimisations

Gaussian Blur, like many blurs, is not super fast. There are some things worth considering:

- Gaussian Blur's slowness scales with the area of the input image
- Gaussian Blur's slowness also scales with the area of the blur kernel

This means that there are a few things that may improve performance:

- Blurring large images reduces performance. Try to limit blurring to smaller images.
- Also, try to limit the blur radius. This can also help performance.
- If you need a larger blur radius, try downscaling the input image, applying a smaller blur, and upscaling again (bilinear scaling works well here).
    - This reduces the area of the input image by (downscale factor)^2, which is brilliant
    - This also reduces the area of the blur kernel by a similar amount.
    - By balancing the downscaling and blur radius, you can achieve optically similar results which are still smooth, but far more performant - at the cost of a small amount of detail.
- If all else fails, try rewriting it in a different language such as C (or, for the clinically insane, try Assembly!)

Of course, you could always try a different algorithm, such as box blur. These algorithms may be faster, depending on your specific use case.
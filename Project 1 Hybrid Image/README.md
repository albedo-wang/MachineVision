# Project 1 Hybrid Image
## 1. Overview
> The goal of this assignment is to write an image filtering function and use it to create hybrid images. Hybrid images are static images that change in interpretation as a function of the viewing distance.The basic idea is that high frequency tends to dominate perception when itis available, but, at a distance, only the low frequency (smooth) part of the signal can be seen.By blending the high frequency portion of one image with the low-frequency portion ofanother, you get a hybrid image that leads to different interpretations at different distances.You will use your own solution to create your own hybrid images

## 2. Implementation Details
- cross_correlation_2d

  > Realize the math function of cross correlation
  ```ruby
  def cross_correlation_2d(kernel, img):
      m, n = np.shape(kernel)
      row, col = np.shape(img)
      pad_num = m // 2
      ans = np.zeros((row-2*pad_num, col-2*pad_num))
      for i in range(pad_num, row-pad_num):
          for j in range(pad_num, col-pad_num):
              cal_matrix = img[i-pad_num:i+pad_num+1, j-pad_num:j+pad_num+1]
              dot_ans = np.multiply(kernel, cal_matrix)
              ans[i-pad_num][j-pad_num] = dot_ans.sum()
      return ans
  ```

- convolve_2d

  > Realize the math function of convolution
  > 
  > One simple but maybe cost much way is to turn the kernel element in inverse order, then use the cross_correlation_2d function
  ```ruby
  def convolve_2d(kernel, img):
      m, n = np.shape(kernel)
      convolve_kernel = np.zeros((m, n))
      for i in range(m):
          for j in range(n):
              convolve_kernel[i][j] = kernel[m-i-1][n-j-1]
      return cross_correlation_2d(convolve_kernel, img)
  ```
  
- gaussian_blur_kernel_2d

  > Generate a 2 dimension gaussian blur kernel
  > 
  > One way is to use the math lib and do the calculation of gaussian distribution (The below code is this way)
  > 
  > Another better way is to use cv2.getGaussianKernel() to generate a 1 dimension vector, then do the vector multiplication to get a 2 dimension kernel
  ```ruby
  def gaussian_blur_kernel_2d(n, sigma):  # n kernel size
      if n % 2 == 0:
          raise Exception("Please enter a odd number for the gaussian filter!")  
          # Here is a error warning set by myself to make sure that can remind me of the kernel size is odd!
      else:
          temp_kernel = np.zeros((n, n))
          center = n // 2 + 1
          for i in range(n):
              for j in range(n):
                  temp_kernel[i][j] = math.exp(-((i - center + 1) ** 2 + (j - center + 1) ** 2) / (2 * sigma ** 2))
          kernel = temp_kernel / temp_kernel.sum()
      return kernel
  ```
  
- low_pass

  > Use the gaussian kernel to do the convolution, then get the low pass frequency image
  > 
  > First, divide the r,g,b channels. Then, use the pad function to pad image. After, do the convolution. Finally, merge them in one RGB image
  ```ruby
  def low_pass(img1, n, sigma):  #img1: input image, n: kernel size, sigma: gaussian kernel sigma
      img = cv2.imread(img1)
      b, g, r = cv2.split(img)
      pad_num = n // 2
      kernel = gaussian_blur_kernel_2d(n, sigma)
      padded_b = np.pad(b, ((pad_num, pad_num), (pad_num, pad_num)), 'constant')
      padded_g = np.pad(g, ((pad_num, pad_num), (pad_num, pad_num)), 'constant')
      padded_r = np.pad(r, ((pad_num, pad_num), (pad_num, pad_num)), 'constant')
      b = convolve_2d(kernel, padded_b)
      g = convolve_2d(kernel, padded_g)
      r = convolve_2d(kernel, padded_r)
      low_pass_img = cv2.merge([b, g, r])
      return low_pass_img.astype(np.uint8)
      # np.uint8 make sure all the convolution element is integer number!
  ```
  
- high_pass

  > Use the low pass image of the last function return. Then, do the subtraction to get the high pass image
  > 
  > First, divide the r,g,b channels. Then, use the pad function to pad image. After, do the convolution. Finally, merge them in one RGB image
  ```ruby
  def high_pass(img2, n, sigma): #img2: input image, n: kernel size, sigma: gaussian kernel sigma
      blur_img = low_pass(img2, n, sigma)
      img = cv2.imread(img2)
      high_pass_img = img - blur_img + 128
      return high_pass_img
      # Due to the low pass image is 0-255 integer, this return do not need the np.uint8!
  ```
  
- hybrid

  > Use the low pass image and high pass image. Then, according to the setted Mix-in ratio to combine them
  > 
  > The original function of hybrid (Don't append any other method)
  ```ruby
  def hybrid(img1, img2, n, sigma, alpha):
      #img1: left image(low pass frequency), img2: right image(high pass frequency), alpha: Mix-in ratio
      low_pass_img = low_pass(img1, n, sigma)
      high_pass_img = high_pass(img2, n, sigma)
      hybrid_img = alpha * low_pass_img + (1 - alpha) * high_pass_img
      hybrid_img = hybrid_img.astype(np.uint8)
      return hybrid_img
  ```
  > Use the global equalizehist
  ```ruby
  def hybrid(img1, img2, n, sigma, alpha):
      #img1: left image(low pass frequency), img2: right image(high pass frequency), alpha: Mix-in ratio
      low_pass_img = low_pass(img1, n, sigma)
      high_pass_img = high_pass(img2, n, sigma)
      hybrid_img = alpha * low_pass_img + (1 - alpha) * high_pass_img
      hybrid_img = hybrid_img.astype(np.uint8)
      b, g, r = cv2.split(hybrid_img)
      global_hist_b = cv2.equalizeHist(b)
      global_hist_g = cv2.equalizeHist(g)
      global_hist_r = cv2.equalizeHist(r)
      return cv2.merge([global_hist_b, global_hist_g, global_hist_r])
  ```
  > Use the local equlizehist
  ```ruby
  def hybrid(img1, img2, n, sigma, alpha):
      #img1: left image(low pass frequency), img2: right image(high pass frequency), alpha: Mix-in ratio
      low_pass_img = low_pass(img1, n, sigma)
      high_pass_img = high_pass(img2, n, sigma)
      hybrid_img = alpha * low_pass_img + (1 - alpha) * high_pass_img
      hybrid_img = hybrid_img.astype(np.uint8)
      b, g, r = cv2.split(hybrid_img)
      clahe = cv2.createCLAHE(1.5, (1, 1))
      local_hist_b = clahe.apply(b)
      local_hist_g = clahe.apply(g)
      local_hist_r = clahe.apply(r)
      return cv2.merge([local_hist_b, local_hist_g, local_hist_r])
  ```

## 3. Test
```ruby
hybrid_ans = hybrid('left.jpg', 'right.jpg', 15, 10, 0.6)
cv2.imwrite('hybrid.jpg', hybrid_ans)
left_img = cv2.imread('left.jpg')
right_img = cv2.imread('right.jpg')
cv2.imshow('Left Image', left_img)
cv2.imshow('Right Image', right_img)
cv2.imshow('Hybrid Image', hybrid_ans)
cv2.waitKey(0)
cv2.destroyAllWindows()
```
> No operation (Original hybrid image)
> 
![Original Image](https://user-images.githubusercontent.com/55045841/136725479-939c9c29-dbba-4895-8298-842f9b6cc5a8.png)

> Global equalizehist
> 
![Global Hist Image](https://user-images.githubusercontent.com/55045841/136725501-d3890d2a-73f5-4b64-8cc8-dceac3f2537b.png)

> Local equalizehise
> 
![Local Hist Image](https://user-images.githubusercontent.com/55045841/136725509-95ef21c8-389b-4dc1-95b9-bf42ecd0219a.png)

> Compare the above three images, we can find that the local equalizehist is the best! Original one and the global equlizehist one are extreme in the contrast ratio.

> So, the final code choose the local equalizehist method.

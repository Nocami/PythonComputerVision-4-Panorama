# PythonComputerVision-4-ImageMosaic
全景图像拼接技术———在同一位置（即图像的照相机位置相同）拍摄的两幅或者多福图像是单应性相关的（p.s上一篇文章中详细介绍了“单应性相关概念”），我们经常使用该约束将很多图像缝补起来，拼成一个大的图像来创建全景图像。在本文中，将要探讨如何创建全景图像。  
## 一.原理介绍 
在进行图像拼接时，首先要解决的是找到图像之间的匹配的对应点。本文采用SIFT算法来实现特征点的匹配，SIFT算法的具体内容参照之前的文章：https://github.com/Nocami/PythonComputerVision-2-SIFT SIFT是很强大的描述子，它能产生很少的错误的匹配，但仍然还是存在错误的对应点。所以需要用一种算法对SIFT算法产生的特征描述符进行剔除误匹配点。
### 1)RANSAC
RANSAC是“RANdom SAmple Consensus”(随机一致性采样)的缩写。RANSAC算法是一种经典的消除误匹配的方法,具有匹配精度高、可靠度强等优点，该方法是用来找到正确模型来拟合带有噪声数据的迭代方法。RANSAC的标准例子：用一条直线拟合带有噪声数据的点集。简单的最小二乘在该例子中可能会失效,但RANSAC可以挑选出正确的点，然后获取能够正确拟合的直线。  
#### 示例
从一组观测数据中找出合适的2维直线。所给出的观测数据中包含正确点和错误点，正确点可以相似的被直线所通过，而错误点远离于直线，分布在其两侧。普通的最小二乘法找不到那条贯穿全部点的直线，因为它会努力的去适应包括错误点在内的所有点。而RANSAC算法能得出一个仅仅用正确点的计算模型，且命中率很高。但尽管如此，它也不能保证100%正确。  
![image](https://github.com/Nocami/PythonComputerVision-4-ImageMosaic/blob/master/images/1.png)![image](https://github.com/Nocami/PythonComputerVision-4-ImageMosaic/blob/master/images/2.png)  
图-包含很多点的数据集&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;图-RANSAC拟合的直线   
### 2)单应性矩阵估计
在任何模型中都可以使用RANSAC模块，这里使用可能的对应点集来自动找到用于全景图像的单应性矩阵--使用SIFT特征自动找到匹配对应，可也使用如下代码：  
~~~python
import sift

featname = ['./images5/'+str(i+1)+'.sift' for i in range(2)] 
imname = ['./images5/'+str(i+1)+'.jpg' for i in range(2)]
l = {}
d = {}
for i in range(2): 
    sift.process_image(imname[i],featname[i])
    l[i],d[i] = sift.read_features_from_file(featname[i])

matches = {}
for i in range(1):
    matches[i] = sift.match(d[i+1],d[i])
~~~
**其中，第一个range（i）中i的值为要拼接的图像个数，第二个为第一个i-1。**  
运行此例代码，我们可以看到，图像中的对应点并不是完全正确，还存在很多错误配对：  
![image](https://github.com/Nocami/PythonComputerVision-4-ImageMosaic/blob/master/images/%E6%99%AF%E6%B7%B1%E5%B0%8FA1.jpg)   
![image](https://github.com/Nocami/PythonComputerVision-4-ImageMosaic/blob/master/images/%E5%AE%A4%E5%86%851.jpg)
![image](https://github.com/Nocami/PythonComputerVision-4-ImageMosaic/blob/master/images/%E6%99%AF%E6%B7%B1%E5%A4%A71.jpg)  
## 二.图像拼接
估计出图像见的单应性矩阵（使用RANSAC算法）后，将所有图像扭曲到一个公共平面上，就完成了一副简单的全景图像。一般的，这个公共平面选择为中心图像的平面，不然会发生大量的形变。因为我们的图像是由照相机水平旋转拍摄成的，所以我们可以使用一个简单的步骤：将中心图像左边或右边的区域填充0，为扭曲图像腾出空间。  
**p.s需要注意的是：**若拼接图为两张，则影响不会很大，中心图像做为平面中心的操作在多福图像拼接时会有明显的效果。  
### 1)代码：
~~~python
from pylab import *
from numpy import *
from PIL import Image

# If you have PCV installed, these imports should work
from PCV.geometry import homography, warp
from PCV.localdescriptors import sift

"""
This is the panorama example from section 3.3.
"""

# set paths to data folder
featname = ['./images5/'+str(i+1)+'.sift' for i in range(2)] 
imname = ['./images5/'+str(i+1)+'.jpg' for i in range(2)]

# extract features and match
l = {}
d = {}
for i in range(2): 
    sift.process_image(imname[i],featname[i])
    l[i],d[i] = sift.read_features_from_file(featname[i])

matches = {}
for i in range(1):
    matches[i] = sift.match(d[i+1],d[i])

# visualize the matches (Figure 3-11 in the book)
for i in range(1):
    im1 = array(Image.open(imname[i]))
    im2 = array(Image.open(imname[i+1]))
    figure()
    sift.plot_matches(im2,im1,l[i+1],l[i],matches[i],show_below=True)


# function to convert the matches to hom. points
def convert_points(j):
    ndx = matches[j].nonzero()[0]
    fp = homography.make_homog(l[j+1][ndx,:2].T) 
    ndx2 = [int(matches[j][i]) for i in ndx]
    tp = homography.make_homog(l[j][ndx2,:2].T) 
    
    # switch x and y - TODO this should move elsewhere
    fp = vstack([fp[1],fp[0],fp[2]])
    tp = vstack([tp[1],tp[0],tp[2]])
    return fp,tp


# estimate the homographies
model = homography.RansacModel() 

fp,tp = convert_points(0)
H_01 = homography.H_from_ransac(fp,tp,model)[0] #im 0 to 1

#fp,tp = convert_points(1)
#H_12 = homography.H_from_ransac(fp,tp,model)[0] #im 1 to 2 

#tp,fp = convert_points(2) #NB: reverse order
#H_32 = homography.H_from_ransac(fp,tp,model)[0] #im 3 to 2 

#tp,fp = convert_points(3) #NB: reverse order
#H_43 = homography.H_from_ransac(fp,tp,model)[0] #im 4 to 3    


# warp the images
delta = 2000 # for padding and translation

im1 = array(Image.open(imname[0]), "uint8")
im2 = array(Image.open(imname[1]), "uint8")
im_12 = warp.panorama(H_01,im1,im2,delta,delta)
#im1 = array(Image.open(imname[0]), "f")
#im_02 = warp.panorama(dot(H_12,H_01),im1,im_12,delta,delta)

#im1 = array(Image.open(imname[3]), "f")
#im_32 = warp.panorama(H_32,im1,im_02,delta,delta)

#im1 = array(Image.open(imname[4]), "f")
#im_42 = warp.panorama(dot(H_32,H_43),im1,im_32,delta,2*delta)


figure()
imshow(array(im_12, "uint8"))
axis('off')
savefig("example1.png",dpi=300)
show()


~~~
此代码段为2图图像拼接，若需要多幅图，只需将其中的注释部分取消即可，图像顺序为自右向左。  
### 2)实例效果
下面我们看看实际效果：  
#### 室外情况下、景深较小：
![image](https://github.com/Nocami/PythonComputerVision-4-ImageMosaic/blob/master/images/%E6%99%AF%E6%B7%B1%E5%B0%8FA1.jpg)  
![image](https://github.com/Nocami/PythonComputerVision-4-ImageMosaic/blob/master/images/%E6%99%AF%E6%B7%B1%E5%B0%8FA2.jpg)  
nice！看起来毫无PS痕迹！  
再看一组同样条件下的照片：  
![image](https://github.com/Nocami/PythonComputerVision-4-ImageMosaic/blob/master/images/%E6%99%AF%E6%B7%B1%E5%B0%8FB1.jpg)  
![image](https://github.com/Nocami/PythonComputerVision-4-ImageMosaic/blob/master/images/%E6%99%AF%E6%B7%B1%E5%B0%8FB2.jpg)  
这一组可以看到明显的拼接缝隙，这是由照片的色差造成的，我们可以看到整体效果还不错。  
#### 室外情况下、景深较大：
![image](https://github.com/Nocami/PythonComputerVision-4-ImageMosaic/blob/master/images/%E6%99%AF%E6%B7%B1%E5%A4%A71.jpg)  
![image](https://github.com/Nocami/PythonComputerVision-4-ImageMosaic/blob/master/images/%E6%99%AF%E6%B7%B1%E5%A4%A72.jpg)  
当物体景深过大时，会产生尺度问题，影响拼接图像的质量，我们可以看到右下角树干有明显瑕疵。  
#### 室内情况、焦距近、图像杂乱：
![image](https://github.com/Nocami/PythonComputerVision-4-ImageMosaic/blob/master/images/%E5%AE%A4%E5%86%851.jpg)  
![image](https://github.com/Nocami/PythonComputerVision-4-ImageMosaic/blob/master/images/%E5%AE%A4%E5%86%852.jpg)  
在这种情况下，算法效果就不是很好了，出现成功拼接的概率大大降低，很容易出现拼接错误等情况。
## 后话：
本文代码运行环境为 python2.7,环境配置以相关文件请访问之前的PythonComputerVision系列文章，链接：https://github.com/Nocami?tab=repositories   
本文所用实例图片为JiMei University。

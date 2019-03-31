# PythonComputerVision-4-ImageMosaic
全景图像拼接技术———在同一位置（即图像的照相机位置相同）拍摄的两幅或者多福图像是单应性相关的（p.s上一篇文章中详细介绍了“单应性相关概念”），我们经常使用该约束将很多图像缝补起来，拼成一个大的图像来创建全景图像。在本文中，将要探讨如何创建全景图像。  
## 原理介绍 
在进行图像拼接时，首先要解决的是找到图像之间的匹配的对应点。本文采用SIFT算法来实现特征点的匹配，SIFT算法的具体内容参照之前的文章：https://github.com/Nocami/PythonComputerVision-2-SIFT SIFT是很强大的描述子，它能产生很少的错误的匹配，但仍然还是存在错误的对应点。所以需要用一种算法对SIFT算法产生的特征描述符进行剔除误匹配点。
### RANSAC
RANSAC是“RANdom SAmple Consensus”(随机一致性采样)的缩写。RANSAC算法是一种经典的消除误匹配的方法,具有匹配精度高、可靠度强等优点，该方法是用来找到正确模型来拟合带有噪声数据的迭代方法。RANSAC的标准例子：用一条直线拟合带有噪声数据的点集。简单的最小二乘在该例子中可能会失效,但RANSAC可以挑选出正确的点，然后获取能够正确拟合的直线。  
#### 示例
从一组观测数据中找出合适的2维直线。所给出的观测数据中包含正确点和错误点，正确点可以相似的被直线所通过，而错误点远离于直线，分布在其两侧。普通的最小二乘法找不到那条贯穿全部点的直线，因为它会努力的去适应包括错误点在内的所有点。而RANSAC算法能得出一个仅仅用正确点的计算模型，且命中率很高。但尽管如此，它也不能保证100%正确。  
![image](https://github.com/Nocami/PythonComputerVision-4-ImageMosaic/blob/master/images/1.png)![image](https://github.com/Nocami/PythonComputerVision-4-ImageMosaic/blob/master/images/2.png)  
图-包含很多点的数据集&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;图-RANSAC拟合的直线   


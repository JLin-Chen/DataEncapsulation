# DataEncapsulation
该文件用于记录在信息隐藏技术课上所学知识及相应的`matlab`实现代码，着重于数字图像信息隐藏的实现。
***  
# 目录
* [基本操作](#基本操作)  
* [不可逆信息隐藏](#不可逆信息隐藏)  
    * 空间域隐写LSB  
    * Writing on Wet Paper    
    * 变换域COX算法  
* [可逆信息隐藏](#可逆信息隐藏)
* [总结](#总结)
* [参考文献](#参考文献)  
***  
## 基本操作
在正文开始之前，先介绍一些matlab上有关图像处理的基本操作。  
读取图像并转为灰度图:
```matlab
I=imread('lena.bmp');
I=rgb2gray(I);
```   
将图像转为双精度：
```matlab
I=im2double(I); %效果较好
I=double(I); %强制类型转换
```
读取txt文档信息：
```matlab
fid=fopen('message.txt','r');
[msg,msg_len]=fread(fid,'ubit1');
```  
导出图像：
```matlab
imwrite(I,'message_hinded.bmp')；
```  
使用的图像：  
![lena](lena.bmp "lena")   

# 不可逆信息隐藏
首先介绍不可逆信息隐藏，它的逻辑与代码实现都相对简单易懂，适合初学者入手。  
在本节中使用万字文章作为秘密信息进行嵌入。为了突出显示嵌入前后的效果对比，以下的实验使用在整个图像中全部嵌入或尽最大可能嵌入的方式。 
## 空间域隐写LSB
LSB又称最低有效位，它对图像像素值的影响最小，可看作冗余信息。若在二进制位中的较高位嵌入，对图像像素影响较大，可能导致图像失真，且较易被察觉嵌入秘密信息。  
![LSB_lena](/img/message_hinded_lena.bmp "不同二进制位嵌入信息效果图")  
我们按顺序遍历图像的每一个像素点，在不大于秘密信息所需长度的地方置LSB位为0并嵌入秘密信息比特。 
这种嵌入方法原理简单，嵌入容量大，嵌入成本低。但这也导致了灰度图中`2i`和`2i+1`处的像素出现“值对”效应。  
![zhidui_lena](/img/LSB_lena.bmp "LSB位嵌入")  
信息嵌入核心代码：  
```matlab
if count<=msg_lenth
   I(i,j)=I(i,j)-mod(I(i,j),2)+msg(count); %LSB位：置0，嵌入
   count=count+1;
end
```
信息提取核心代码：  
```matlab
if count<=msg_lenth
   w(count)=mod(I(i,j),2); %提取最低位信息
   count=count+1;
end
```
因为LSB算法中需要把LSB位置为0，在不使用位置图的情况下无法将图片复原，所以该方法是不可逆的信息隐藏算法。  
## "Writing on Wet Paper"
在上述方法中，由于秘密信息是按照顺序嵌入图像中的，嵌入和提取过程都较为简单，这也易导致秘密信息的意外泄漏。
因此`Jessica et al.`在2005年于IEEE上发表[1]提出一种随机嵌入的方法，该方法也被称作“湿码通信”,它使得信息较不容易被窃取。  

我们使用图像的红色图层来隐藏信息：
```matlab
image1=Hide_image(:,:,1);
```  
在红色图层中，我们观察到仍存在一定的“值对”效应。但是由于WM方法是随机像素点嵌入，牺牲了一定的嵌入容量，所以“值对”效应相对传统的LSB位嵌入有一定的改善。  
![WM_lena_red](/img/WM_Red.bmp)  
信息嵌入核心代码：
```matlab
%寻找嵌入点
[row,col]=randinterval(image1,msg_len,1996);   % 1996是随机数种子
%嵌入信息
for i=1:msg_len
    image1(row(i),col(i))=image1(row(i),col(i))-mod(image1(row(i),col(i)),2)+msg(i,1);
end
%将图像融合
Hide_image(:,:,1)=image1;
Hide_image=uint8(Hide_image);
```
信息提取核心代码：
```matlab
%调用同一个函数再次得到所嵌入的位置的信息
[row,col]=randinterval(Picture_R,msg_len,1996);
%提取信息
for i=1:msg_len
    if bitand(Picture(row(i),col(i)),1)==1 %像素与比特位进行按位与,提出最后一位秘密信息比特
        fwrite(fid,1,'ubit1');
    else
        fwrite(fid,0,'ubit1');
    end
end
```  
这里，我们用到了一个函数`randinterval`，它的作用是返回随机数向量，该向量表明了应在何处嵌入信息。  
当红色图层与其他图层融合后，我们把图像转为灰度图来观察是否存在“值对”效应。
通过实验，我们发现，虽然嵌入前后的直方图存在差别，但并没有明显的值对效应，信息隐藏的效果较传统LSB嵌入更好。  
![WM_RGB_lena](/img/WM_RGB.bmp)  
该方法牺牲了可嵌入的信息容量、增加了算法复杂度，需要收发双方均知道随机数种子的数值大小(即拥有密钥)，在一定程度上提高了信息的保密性。  
 

## 变换域COX算法  
上述两种方法都是在空间域进行信息隐藏，即直接在载体数据上加载水印信息。接下来我们使用的是经过DCT变换后的信息隐藏方法。该方法也称为NEC算法或基于扩频技术的算法，它[2]于1997年被提出，收录于IEEE。    
为解决信息嵌入的不可见性和鲁棒性之间的矛盾，通常将信息嵌入AC低频系数中。  
alpha的值即为水印的强度，值越大则嵌入的信息越不容易因受到扰动而被篡改。但alpha的值如果设置得过大则会对图像产生一定的影响：  
![Cox_lena_alpha](/img/Cox_lena.bmp "不同alpha值的嵌入效果")  
信息嵌入函数：  
```matlab 
function [ J ] = Cox_embed( I, W, alpha, N ) %图像，水印，水印的强度， 水印的长度
[m,n]=size(I);
if (m*n<N)
    error('载体图像过小');
end
P=double(I)/255; %双精度，归一化
DCTI=dct2(P);
[index_row,index_col]=FindNLargest(abs(DCTI),N);
for i=1:N
    DCTI(index_row(i),index_col(i))=DCTI(index_row(i),index_col(i))*(1+alpha*W(i)); %嵌入信息
end
J=idct2(DCTI); %反变换，J为含水印的图像
imwrite(J,'secret_lena.bmp');
end
```  
信息提取函数：  
```matlab
function [ Watermark_Extracted ] = Cox_extract( I, J, alpha, N  ) %原图像，含水印图像，水印强度， 水印长度
[m,n]=size(I);
[x,y]=size(J);
if ((m~=x)|(n~=y))
    error('图像大小不一致');
end
DCTI=dct2(I);
DCTJ=dct2(J);
[index_row,index_col]=FindNLargest(abs(DCTI),N);
for i=1:N
    Watermark_Extracted(i)=(DCTJ(index_row(i),index_col(i))/DCTI(index_row(i),index_col(i))-1)/alpha;
end
Watermark_Extracted=int8(abs(Watermark_Extracted)); %防止产生未知错误，转换为整型
end
```   
# 可逆信息隐藏
在某些应用场景下，我们希望在提取完秘密信息后能够恢复载体图像。下面将介绍经典的直方图平移算法以及基于该算法提出的一些改进方法。   

## 直方图平移算法  
对于一个图像，以横坐标为图像中每个像素点的值，纵坐标为每个像素值所出现的频率，所得的即为该图像的直方图。  
直方图平移算法就是根据图像的直方图中存在峰值点和零值点的特征而设计的一种可逆信息隐藏算法。找到距直方图峰值点最近的零值点，把二者之间的像素进行平移，便能在峰值点左右得到嵌入空间，在峰值点进行嵌入时，嵌入空间最大。多对峰值点嵌入的方法也能有效提高嵌入信息的容量。  
以lena灰度图为例，在使用直方图平移的嵌入方法下，得到的平移前、平移后和嵌入后的直方图：  
![histogram_lena](/img/histogram_lena.bmp "histogram")  
首先,先得到图像直方图数据的数组表示:  
```matlab
[counts,grayvalue]=imhist(image); 
```  
然后，找到峰值点和零值点:  
```matlab
Peak_point=max(grayvalue(find(counts==max(counts))));  %find返回最大值对应的横坐标，grayvalue返回灰度值对应的值，最外层max防止有多个峰值的情况，只取简单的一个最大值 
x=grayvalue(find(counts==min(counts))); %找到最小值的点，但可能是多值
y=abs(x-Peak_point);  %得到一组值
Zero_point=x(find(y==min(y)));
```  
进行直方图的平移,使得峰值点附近出现嵌入空间：    
```matlab
for i=1:m
    for j=1:n
        if(Peak_point>Zero_point)
            if((image(i,j)>Zero_point)&&(image(i,j)<Peak_point))
                image(i,j)=image(i,j)-uint8(1);
            end
        else
            if((image(i,j)<Zero_point)&&(image(i,j)>Peak_point))
                image(i,j)=image(i,j)+uint8(1); 
            end
        end
    end
end 
```  
嵌入信息：  
```matlab
%嵌入信息
W_index=1;
if(Peak_point<Zero_point)
    for i=1:m
        if W_index>length(W) 
            break; %信息嵌入完毕跳出循环
        end
        for j=1:n
            if W_index<=length(W)
                if image(i,j)==Peak_point
                    if W(W_index)==1
                        image(i,j)=image(i,j)+1;
                        W_index=W_index+1;
                    else 
                        image(i,j)=image(i,j);
                        W_index=W_index+1;
                    end
                end
            end
        end
    end
else
    for i=1:m
        if W_index>length(W) 
            break; %信息嵌入完毕跳出循环
        end
        for j=1:n
           if W_index<=length(W)
                if image(i,j)==Peak_point
                    if W(W_index)==1
                        image(i,j)=image(i,j)-1;
                        W_index=W_index+1;
                    else 
                        image(i,j)=image(i,j);
                        W_index=W_index+1;
                    end
                end
           end
        end
    end
end                
```   
提取信息：  
```matlab
for i=1:m
    if k>N
        break; %信息已提取完毕
    end
    for j=1:n
        if(Peak_point<Zero_point)
            if k>N
                break;
            end
            if T(i,j)==Peak_point
                Wa(k)=0;
                k=k+1;
            elseif T(i,j)==(Peak_point+1)
                Wa(k)=1;
                k=k+1;
            end
        else 
            if k>N
                break;
            end
            if T(i,j)==Peak_point
                Wa(k)=0;
                k=k+1;
            elseif T(i,j)==(Peak_point-1)
                Wa(k)=1;
                k=k+1;
            end
        end
    end
end
```   
图像复原：
```matlab
for i=1:m
    for j=1:n
        if(Peak_point>Zero_point)
            if((T(i,j)>=Zero_point)&&(T(i,j)<Peak_point))  %>=确保包括零值点都能得到复原
                T(i,j)=T(i,j)+uint8(1);
            end
        else
            if((T(i,j)<=Zero_point)&&(T(i,j)>Peak_point))
                T(i,j)=T(i,j)-uint8(1); 
            end
        end
    end
end 
```  
但是，直方图平移算法可能出现没有零值点的问题，在没有零值点的情况下，上述算法将无法实现。 
若图像没有零值点，则需寻找最低直方，记录直方中的像素值以及原图中等于该像素值得像素所在的位置，然后将该直方置为０。虽然该方法解决了直方图没有零值点的问题，但是由于需要记录位置图，会造成一定的信息冗余，且占用有效隐藏容量。 

## Border-Following直方图平移算法  
为了解决图像可能没有零值点的问题，Border－Following方法被提出。该方法通过把图像分成多个小块，使得零值点有更高的概率出现。这种方法是对经典的直方图平移算法的一种改进和补充。  
但是，由于直方图平移算法在平移完零点后，直方图中会有很明显的峰值点旁边的值的缺失，因此图像隐藏有秘密信息这一事实较易被发现。为解决这个问题以及直方图算法嵌入容量受峰值点影响较大的情况，一种新的像素差直方图修改算法产生了。  
但在这种应用场景下，需要传递的峰值点和零值点的信息增多了

## 像素差直方图修改算法  
在上述直方图平移算法中，因为将零值点平移至峰值点旁边，
这导致了在图像的直方图中产生了肉眼极易察觉的凹陷，这也导致他人能容易察觉这是一幅嵌入了秘密信息的图像。
而且，在直方图平移算法中，嵌入信息量的大小受峰值点影响，可能存在嵌入容量过小的情况。  
为了解决包含上述问题在内的一系列问题像素差直方图修改算法产生，该方法也被称为“张方法”。  
该方法的二叉树的高度最高只能达到5，但是二叉树的高度越高，对图像像素造成的改变也就越大，图像失真越严重。  
![HM_PD_lena](/img/HM_PD_lena.bmp "lena")  
由于实验选用的lena图像在靠近0和靠近255处的像素值不多，实验效果不明显，所以我们另选了一张图：  
![HM_PD_yuan2](/img/HM_PD_yuan_2.bmp "yuan")二叉树高度为2  ![yuan2](/img/yuan_2.bmp)  
![HM_PD_yuan3](/img/HM_PD_yuan_3.bmp "yuan")二叉树高度为3  ![yuan2](/img/yuan_3.bmp)  
![HM_PD_yuan5](/img/HM_PD_yuan_5.bmp "yuan")二叉树高度为5  ![yuan2](/img/yuan_5.bmp)  

## 高保真直方图平移算法  
张方法虽然解决了直方图平移后图像中存在凹陷的问题，但是却存在某些像素值改动过大的问题。为了解决凹陷问题和改动多大问题，Tang et al.提出了一种高保真的直方图平移算法[[3]](#参考文献)。  
它不仅解决了直方图中存在凹陷的问题，因为其存在像素补足，所以它还能使的在嵌入秘密信息的同时对图像的改动更小。  
但又因为其一系列的算法要求，导致能够满足嵌入条件的像素点可能比较少，导致嵌入容量小的问题。  

# 总结  




















# 参考文献
  [1]J. Fridrich, M. Goljan, P. Lisonek and D. Soukal, "Writing on wet paper," in IEEE Transactions on Signal Processing, vol. 53, no. 10, pp. 3923-3935, Oct. 2005, doi: 10.1109/TSP.2005.855393.  
  [2]I. J. Cox, J. Kilian, F. T. Leighton, and T. Shamoon. 1997. Secure spread spectrum watermarking for multimedia. <i>Trans. Img. Proc.</i> 6, 12 (December 1997), 1673–1687. DOI:https://doi.org/10.1109/83.650120  
  [3]https://www.researchgate.net/publication/348095095_Reversible_Data_Hiding_Algorithm_with_High_Imperceptibility_Based_on_Histogram_Shifting

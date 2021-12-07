# DataEncapsulation
该文件用于记录在信息隐藏技术课上所学知识及相应的`matlab`实现代码，着重于数字图像信息隐藏的实现。
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

# 不可逆信息隐藏
首先介绍不可逆信息隐藏，它的逻辑与代码实现都相对简单易懂，适合初学者入手。
## 空间域隐写LSB
LSB又称最低有效位，它对图像像素值的影响最小，可看作冗余信息。若在二进制位中的较高位嵌入，对图像像素影响较大，可能导致图像失真，且较易被察觉嵌入秘密信息。  
我们按顺序遍历图像的每一个像素点，在不大于秘密信息所需长度的地方置LSB位为0并嵌入秘密信息比特。  
信息嵌入核心代码：  
```matlab
for i=1:m
    for j=1:n
        if count<=msg_len
            I(i,j)=I(i,j)-mod(I(i,j),2)+msg(count); %LSB位：置0，嵌入
            count=count+1;
        end
    end
end
```
信息提取核心代码：  
```matlab
for i=1:m
    for j=1:n
        if count<=msg_lenth
            w(count)=mod(I(i,j),2); %提取最低位信息
            count=count+1;
        end
    end
end
```
因为LSB算法中需要把LSB位置为0，在不使用位置图的情况下无法将图片复原，所以该方法是不可逆的信息隐藏算法。  
## "Writing on Wet Papper"
在上述方法中，由于秘密信息是按照顺序嵌入图像中的，嵌入和提取过程都较为简单，这也易导致秘密信息的意外泄漏。
因此`Jessica et al.`在2005年于IEEE上发表论文[1]提出一种随机嵌入的方法，该方法也被称作“湿码通信”,它使得信息较不容易被窃取。  
这里，我们用到了一个函数`randinterval`，它的作用是返回随机数向量，该向量表明了应在何处嵌入信息。  
randinterval:  
```matlab
function [ row,col] = randinterval( matrix,count,key ) %key是随机数种子,输出信息隐藏处的向量
[m,n]=size(matrix); %显示矩阵的大小
interval1=floor(m*n/count)+1; %向下取整
interval2=interval1-2;
if interval2 == 0
    error('载体太小不能将信息隐藏进去');
end
rand('seed',key);
a=rand(1,count);
row=zeros([1,count]); %把1到第count列初始化为0
col=zeros([1,count]);
r=1;
c=1;
row(1,1)=r; %1行n列的矩阵，即向量
col(1,1)=c;
for i=2:count
    if a(i)>=0.5
        c=c+interval1;
    else
        c=c+interval2;
    end
    if c>n
       r=r+1;
       if r>m
           error('载体太小不能嵌入');
       end
       c=mod(c,n);
       if c==0
           c=c+1;
       end
    end
    row(1,i)=r;
    col(1,i)=c;

end
```  
我们使用图像的红色图层来隐藏信息：
```matlab
image1=Hide_image(:,:,1);
```
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
    if bitand(Picture(row(i),col(i)),1)==1 %像素与比特位进行按位与.提出最后一位秘密信息比特
        fwrite(fid,1,'ubit1');
    else
        fwrite(fid,0,'ubit1');
    end
end
```
















# 参考文献
  [1]J. Fridrich, M. Goljan, P. Lisonek and D. Soukal, "Writing on wet paper," in IEEE Transactions on Signal Processing, vol. 53, no. 10, pp. 3923-3935, Oct. 2005, doi: 10.1109/TSP.2005.855393.

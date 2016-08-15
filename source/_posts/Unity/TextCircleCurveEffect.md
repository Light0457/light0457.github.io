---
title: Unity UI文字绕圆心扭曲效果
tags: 
- Unity     
- Image Effect   
- Text    
- Circular   
---

# 需求   
使文字绕着某一圆心排列。（图片效果后面补齐）  

# 实现  
### 输入:  
* w, 文字总宽度   
* h, 文字总高度
* n, 圆心离文字顶部的距离(按n倍h来计算),因此圆心离文字底部的距离为(n+1)*h
* inVerts = [{x1,y1},{x2,y2}...], 文字顶点序列  
* anchor, 圆心延y轴投影到文字中的x轴上的归一化位置[0,1]

### 输出: 
* outVerts = [{x1',y1'},{x2',y2'}...],调整完位置后的顶点 

### 算法描述:  
1. 设定圆心在(0,0)点；(方便进行坐标转换)
2. 记nh为内圆半径innerRadius,定位center到(0,-innerRadius)处,记下此处偏移量offset
3. 针对每个在inVerts中的顶点inVert:  
    3.1 根据center偏移顶点到圆心坐标系中,记转换后的顶点为vert;   
    3.2 当前这个顶点距圆心的距离即为abs(vert.y),记为raidus;  
    3.3 算出vert与center的x距离,此距离即是在圆上离center的弧长,记为arcLen(此处应保留正负符号,方便角度转换);  
    3.4 根据圆弧长公式
    ```
    圆弧长arcLen = PI * R(半径) * a(角度) / 180
    ```
    可以算出,从center位置绕着圆心转到vert在圆边上对应位置处需要转过的角度为
    ```
    角度位置angleCV = arcLen * 180 / (PI * innerRadius)
    ``` 
    3.5 将angleCV转换成极座标系里面的角坐标
    ```
    极座标系的角坐标angle = 90 - angleCV 
    ``` 
    (PS: 此处angleCV因为arcLen保持正负符号的关系,它也保持着正负符号)
    3.6 换算出vert转换后的坐标:
    ```
    x = raidus * cos(angle)
    y = raidus * sin(angle)
    ```
    3.7 偏移顶点回到回来的坐标系当中
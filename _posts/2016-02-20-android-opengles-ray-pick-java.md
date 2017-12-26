---
layout: default
title: Android OpenGL ES拾取（使用Java）
date: 2016-02-20 11:50:00
categories: blog
tags: [Android, Java, "OpenGl ES", "Ray Pick", "3D"]
description: Android OpenGL ES拾取（使用Java）
---


## 第一步：建立拾取射线

核心函数是`GLU.gluUnProject`，原型如下
```java
static int gluUnProject(float winX, float winY, float winZ, float[] model, int modelOffset, float[] project, int projectOffset, int[] view, int viewOffset, float[] obj, int objOffset)
```
winZ取0得到射线近点，winZ取1得到远点。模型矩阵（model）和投影矩阵（project）比较麻烦，由于ES版本不提供获取模型和投影矩阵的方法，因此通过自行跟踪矩阵变换的方式来解决。
具体实现由`MatrixTrackingGL`和`MatrixGrabber`以及`MatrixStack`三个类协作提供，这三个类可以在`\samples\android\ApiDemos\src\com\example\android\apis\graphics\spritetext\`文件夹下找到。 

视图矩阵（参数int[] view）是在render的`onSurfaceChanged`时提供的长宽构建的，如下
```java
@Override
public void onSurfaceChanged(GL10 gl, int width, int height) {
    viewPort = new int[]{0,0,width,height}
}
```
最后的obj参数是获取到的结果，obj参数被赋值为4元素数组，要用前三个除以第四个得到最终结果。下面是获取射线的代码
```java
public class Ray {
    public Ray(MatrixGrabber matrixGrabber,int[] viewPort,float xTouch, float yTouch){
        float[] temp = new float[4];

        float winy =(float)viewPort[3] - yTouch;
        int result = GLU.gluUnProject(xTouch, winy, 1f, matrixGrabber.mModelView, 0, matrixGrabber.mProjection, 0, viewPort, 0, temp, 0);
        if(result == GL10.GL_TRUE){
            farCoords[0] = temp[0] / temp[3] * Cube3.one;
            farCoords[1] = temp[1] / temp[3] * Cube3.one;
            farCoords[2] = temp[2] / temp[3] * Cube3.one;
        }
        result = GLU.gluUnProject(xTouch, winy, 0, matrixGrabber.mModelView, 0, matrixGrabber.mProjection, 0, viewPort, 0, temp, 0);
        if(result == GL10.GL_TRUE){
            nearCoords[0] = temp[0] / temp[3] * Cube3.one;
            nearCoords[1] = temp[1] / temp[3] * Cube3.one;
            nearCoords[2] = temp[2] / temp[3] * Cube3.one;
        }
    }

    public float[] farCoords = new float[3];
    public float[] nearCoords = new float[3];
}
```
其中`Cube3.one=0x10000`，因为我的数据是整数的，乘以0x10000将float转化为int。

## 第二步：获取焦点

判断一个三角形和射线是否有交点，这个过程用到内积、外积等向量运算，是网上的人用原c代码转Java
```java
    /**
     * 检测射线与三角形相交
     * @param R 射线
     * @param T 三角形
     * @param I 焦点
     * @return -1:无法构造三角形（三点一线，三点重合）
     *         0:无交点
     *         1:相交于I
     *         2:射线与三角形共面
     */
    public static int intersectRayAndTriangle(Ray R, Triangle T, float[] I){
        float[]    u, v, n;             // triangle vectors
        float[]    dir, w0, w;          // ray vectors
        float     r, a, b;             // params to calc ray-plane intersect
        // get triangle edge vectors and plane normal
        u =  Vector.minus(T.V1, T.V0);
        v =  Vector.minus(T.V2, T.V0);
        n =  Vector.crossProduct(u, v);             // cross product
        if (Arrays.equals(n, new float[]{0.0f, 0.0f, 0.0f})){           // triangle is degenerate
            return -1;                 // do not deal with this case
        }
        dir =  Vector.minus(R.farCoords, R.nearCoords);             // ray direction vector
        w0 = Vector.minus(R.nearCoords, T.V0);
        a = - Vector.dot(n,w0);
        b =  Vector.dot(n,dir);
        if (Math.abs(b) < SMALL_NUM) {     // ray is parallel to triangle plane
            if (a == 0){                // ray lies in triangle plane
                return 2;
            }else{
                return 0;             // ray disjoint from plane
            }
        }
        // get intersect point of ray with triangle plane
        r = a / b;
        if (r < 0.0f){                   // ray goes away from triangle
            return 0;                  // => no intersect
        }
        float[] tempI =  Vector.addition(R.nearCoords,  Vector.scalarProduct(r, dir));           // intersect point of ray and plane
        I[0] = tempI[0];
        I[1] = tempI[1];
        I[2] = tempI[2];
        // is I inside T?
        float    uu, uv, vv, wu, wv, D;
        uu =  Vector.dot(u,u);
        uv =  Vector.dot(u,v);
        vv =  Vector.dot(v,v);
        w =  Vector.minus(I, T.V0);
        wu =  Vector.dot(w,u);
        wv = Vector.dot(w,v);
        D = (uv * uv) - (uu * vv);
        // get and test parametric coords
        float s, t;
        s = ((uv * wv) - (vv * wu)) / D;
        if (s < 0.0f || s > 1.0f)        // I is outside T
            return 0;
        t = (uv * wu - uu * wv) / D;
        if (t < 0.0f || (s + t) > 1.0f)  // I is outside T
            return 0;

        return 1;                      // I is in T
    }
```
如果返回1，那么I就是交点。其中用到的向量工具类：
```java
public class Vector {
    // dot product 内积
    public static float dot(float[] u,float[] v) {
        return ((u[X] * v[X]) + (u[Y] * v[Y]) + (u[Z] * v[Z]));
    }
    public static float[] minus(float[] u, float[] v){
        return new float[]{u[X]-v[X],u[Y]-v[Y],u[Z]-v[Z]};
    }
    public static float[] addition(float[] u, float[] v){
        return new float[]{u[X]+v[X],u[Y]+v[Y],u[Z]+v[Z]};
    }
    //scalar product 缩放
    public static float[] scalarProduct(float r, float[] u){
        return new float[]{u[X]*r,u[Y]*r,u[Z]*r};
    }
    // cross product 外积
    public static float[] crossProduct(float[] u, float[] v){
        return new float[]{(u[Y]*v[Z]) - (u[Z]*v[Y]),(u[Z]*v[X]) - (u[X]*v[Z]),(u[X]*v[Y]) - (u[Y]*v[X])};
    }
    // length 长度
    public static float length(float[] u){
        return (float) Math.abs(Math.sqrt((u[X] *u[X]) + (u[Y] *u[Y]) + (u[Z] *u[Z])));
    }

    public static final int X = 0;
    public static final int Y = 1;
    public static final int Z = 2;
}
```
全部源代码放在[github](https://github.com/x6uderek/rubik)上面，是一个魔方程序，可以用手指拾取魔方块进行旋转。
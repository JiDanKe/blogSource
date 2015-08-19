title: "从 Texture Maping 理解可编程的图形管线"
date: 2015-08-19 14:15:00
categories: 计算机图形
tags: 计算机图形
---

#Texture Mapping

在 Unity 3D 引擎中要渲染一个3D模型需要这样一些基本组件：`Mesh`、`Material`和`Shader`。其中`Mesh`能够表示模型的几何形状，在有了模型的几何形状之后游戏引擎就可以把`Material`渲染到模型表面。`Material`的表现效果则可以由着色器也就是`Shader`来控制。

如果模型表面只需要渲染纯色，那么我们只需在着色器中得的块作色器（fragment shader）中返回模型表面每个像素点需要的颜色即可；但是如果想要把事先制作好的图片（也就是贴图）渲染到模型上就要相对复杂一些，因为模型表面每个像素点的颜色我们需要从贴图上获取，要准确地获取到颜色必须要把模型表面的像素点和贴图上的像素点对应起来，这个过程就叫做 Texture Mapping。

在着色器的层面上 Texture Mapping 可以用下面这个简单的作色器表示：

```
Shader "Cg shader with single texture" {
   Properties {
      _MainTex ("Texture Image", 2D) = "white" {} 
         // 要用来渲染模型的贴图
   }
   SubShader {
      Pass {    
         CGPROGRAM
 
         #pragma vertex vert  
         #pragma fragment frag 
 
         uniform sampler2D _MainTex;    
 
         struct vertexInput {
            float4 vertex : POSITION;
            float4 texcoord : TEXCOORD0;
         };
         struct vertexOutput {
            float4 pos : SV_POSITION;
            float4 tex : TEXCOORD0;
         };
 
         vertexOutput vert(vertexInput input) 
         {
            vertexOutput output;
 
            output.tex = input.texcoord;
               // 顶点在贴图坐标的坐标，也就是 UV 坐标 
			   
            output.pos = mul(UNITY_MATRIX_MVP, input.vertex);
            return output;
         }
		 
         float4 frag(vertexOutput input) : COLOR
         {
            return tex2D(_MainTex, input.tex.xy);   
               // 根据模型表面像素点的 UV 坐标获取贴图上的颜色进行渲染
         }
 
         ENDCG
      }
   }

}
```

从上面的代码中我们可以看出，在顶点着色器中我们只是输入了顶点的 UV 坐标，但是在块着色器中却可以通过模型表面像素点的 UV 坐标来获取了贴图上颜色，至于这些像素点的 UV 坐标从何而来我们从代码上是看不出来的。原因是模型表面像素点的 UV  坐标的确定并不是通过在着色器中编程实现的，这个过程是由图形管线自动的完成的。那么这个过程是在什么时候完成的呢？这就需要去了解一下关于图形管线（Graphics Pipeline）的相关知识了。

<!-- more -->

#可编程的图形管线

##图形管线的工作流程

图形管线的工作流程可以表示为如下步骤：

Vertex Data -> Vertex Shader -> Primitive -> Rasterization -> Fragment Shader -> Per-Fragment Operations -> Framebuffer

在这些过程中我们可以编程的地方只有 Vertex Shader 和 Fragment Shader。在 Vertex Shader 中我们可以对由3D模型提供的 Vertex（顶点）Data 进行一些简单的处理和变化；在 Fragment Shader 中可以对模型上每一个像素点进行处理。而模型上像素点的 UV 坐标的确定则是由 Rasterization（光栅化） 来完成。

###光栅化

3D模型的表面是由一系列的几何图形组合而成的，这些图形的顶点就是由模型提供的顶点数据。光栅化会完成以下两部分工作：

1. 决定每个由顶点组成的图形中包含的像素点。完成这个工作主要依据的原则就是：每个像素都要被覆盖到，而且每个图形之间都不会覆盖到相同的像素点。

2. 使用差值法确定图形中每个像素中对应的在贴图坐标系中的坐标（UV 坐标）和其他一些参数。在顶点着色器中，我们会把顶点坐标转化为贴图坐标系中的坐标，而像素点的 UV 坐标则是在光栅化时通过差值来获取的。差值的方法有很多种，在这里就不详述了。

在这些工作完成之后，通过光栅化得到的信息都会作为输入供块着色器使用。对应到上面的着色器代码，`input.tex.xy` 这个数据就是经过了光栅化处理了的数据，这样模型上的像素点就可以很好地和贴图上的像素点对应起来了。也就完成了 Texture Mapping。

#参考资料
* [Textured Spheres](https://en.wikibooks.org/wiki/Cg_Programming/Unity/Textured_Spheres)
* [Rasterization](https://en.wikibooks.org/wiki/Cg_Programming/Rasterization)
* [Programmable Graphics Pipeline](https://en.wikibooks.org/wiki/Cg_Programming/Programmable_Graphics_Pipeline)



---
layout: post
category: "dev"
title: "ShaderToy系列之一: 炫酷的海洋渲染Seascape"
tags: "Graphics"
---

# 关于ShaderToy

[Shadertoy](https://www.shadertoy.com/)是GLSL图形程序员的一个在线playground，基于WebGL的便携和跨平台特性，让本来需要繁琐开发环境的shader开发变得简单，随时随地都可以创建，修改以及分享自己的写的shader。在ShaderToy上能随意地发现很多乍看不可思议的效果，比如：

* [渲染一颗恒星](https://www.shadertoy.com/view/4dXGR4)
* [体积云](https://www.shadertoy.com/view/Mt3GWs)
* [超级马里奥!](https://www.shadertoy.com/view/Msj3zD)

# ShaderToy实现

乍一看这么炫酷的效果，其原理一定很深奥吧！其实不然，shadertoy本身的原理十分简单，我们只需要做极少的工作就能把shadertoy上的shader源码原封不动的拿到我们自己的GL项目里面来(参考[github](https://github.com/rechardchen/play/blob/master/graphics/shadertoy.h))。

因为本身没有数据输入，所以所有的计算都是在fragment shader里面完成的。ray tracing, procedure sdf, volumn rendering是这里常用的手段。

# 炫酷的海洋渲染Seascape

<p style="text-align: center;">
<img src="/img/seascape.png" style="width:85%">
</img>
<i style="display:block;">seascape</i>
</p>

[Seascape](https://www.shadertoy.com/view/Ms2SD1)是我在shadertoy上看到的第一个令人称奇的特效，只用了短短150行shader代码。下面来分析一下，

{% codeblock [lang:Cpp] [linenos:true] %}
// main
void mainImage( out vec4 fragColor, in vec2 fragCoord ) {
	vec2 uv = fragCoord.xy / iResolution.xy;
    uv = uv * 2.0 - 1.0;
    uv.x *= iResolution.x / iResolution.y;    
    float time = iGlobalTime * 0.3 + iMouse.x*0.01;
        
    // ray
    vec3 ang = vec3(sin(time*3.0)*0.1,sin(time)*0.2+0.3,time);    
    vec3 ori = vec3(0.0,3.5,time*5.0); //原点
    vec3 dir = normalize(vec3(uv.xy,-2.0)); dir.z += length(uv) * 0.15;
    dir = normalize(dir) * fromEuler(ang);//欧拉角
    
    // tracing
    vec3 p;
    heightMapTracing(ori,dir,p);
    vec3 dist = p - ori;
    vec3 n = getNormal(p, dot(dist,dist) * EPSILON_NRM);
    vec3 light = normalize(vec3(0.0,1.0,0.8)); 
             
    // color
    vec3 color = mix(
        getSkyColor(dir),
        getSeaColor(p,n,light,dir,dist),
    	pow(smoothstep(0.0,-0.05,dir.y),0.3));
        
    // post
	fragColor = vec4(pow(color,vec3(0.75)), 1.0);
}
{% endcodeblock %}

mainImage是shadertoy里的main函数，fragColor是像素着色器的色彩输出。






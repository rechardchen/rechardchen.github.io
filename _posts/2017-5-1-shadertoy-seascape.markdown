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
<img src="/img/seascape.PNG" style="width:85%"/>
<i style="display:block;">seascape</i>
</p>

[Seascape](https://www.shadertoy.com/view/Ms2SD1)是我在shadertoy上看到的第一个令人称奇的特效，只用了短短150行shader代码。下面来分析一下，

{% highlight c linenos %}
// main
void mainImage( out vec4 fragColor, in vec2 fragCoord ) {
    vec2 uv = fragCoord.xy / iResolution.xy;
    uv = uv * 2.0 - 1.0;
    uv.x *= iResolution.x / iResolution.y;    
    float time = iGlobalTime * 0.3 + iMouse.x*0.01;
        
    // ray
    vec3 ang = vec3(sin(time*3.0)*0.1,sin(time)*0.2+0.3,time);    
    vec3 ori = vec3(0.0,3.5,time*5.0); //原点
    vec3 dir = normalize(vec3(uv.xy,-2.0)); dir.z += length(uv) * 0.15;//广角
    dir = normalize(dir) * fromEuler(ang);//dir = normalize(dir);去掉euler angle便于研究
    
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
{% endhighlight %}

mainImage是shadertoy里的main函数，fragColor是像素着色器的色彩输出。所有shadertoy着色器都需要用的两个uniform变量是iResolution和iGlobalTime，前者配合fragCoord可以得到像素位置，iGlobalTime用来计时。line10的原点表明RayTrace的原点是沿着z轴向画面外移动。**可以把12行的fromEuler注释掉，这样的话去掉镜头摇摆，可以更加方便我们的研究**。

Trace阶段，首先heightMapTracing得到dir方向上的视线终点p，计算p点处的法线方向n。这里涉及到海洋渲染的核心函数如下(**sea_octave的算法还没有完全理解，但是直接拿到项目中还是可以的**)。

{% highlight c linenos %}
float map_detailed(vec3 p) {
    float freq = SEA_FREQ;//SEA_FREQ用来控制水波的频率
    float amp = SEA_HEIGHT;//SEA_HEIGHT控制浪高
    float choppy = SEA_CHOPPY; //SEA_CHOPPY控制波涛宽度	
    vec2 uv = p.xz; uv.x *= 0.75;
    
    float d, h = 0.0;    
    for(int i = 0; i < ITER_FRAGMENT; i++) {//浪的位置需要在几个位置上做采样叠加
        d = sea_octave((uv+SEA_TIME)*freq,choppy);
        d += sea_octave((uv-SEA_TIME)*freq,choppy);
        h += d * amp;        
        uv *= octave_m; //uv变换，下一个采样位置
        freq *= 1.9; amp *= 0.22;
        choppy = mix(choppy,1.0,0.2);
    }
    return p.y - h;
}

float sea_octave(vec2 uv, float choppy) {
    uv += noise(uv);//加上noise使海面看上去更加真实
    vec2 wv = 1.0-abs(sin(uv));
    vec2 swv = abs(cos(uv));    
    wv = mix(wv,swv,wv);
    return pow(1.0-pow(wv.x * wv.y,0.65),choppy);
}

float noise( in vec2 p ) {
    vec2 i = floor( p );
    vec2 f = fract( p );	
    vec2 u = f*f*(3.0-2.0*f);
    return -1.0+2.0*mix( mix( hash( i + vec2(0.0,0.0) ), 
                     hash( i + vec2(1.0,0.0) ), u.x),
                mix( hash( i + vec2(0.0,1.0) ), 
                     hash( i + vec2(1.0,1.0) ), u.x), u.y);
}
{% endhighlight %}

用我们的老朋友Matlab看一下TDM的这个海洋函数，

<p style="text-align: center;">
<img src="/img/sea_octave.png" style="width:85%"/>
<i style="display:block;">sea_octave</i>
</p>

最后是着色阶段，getSkyColor就是很简单的视线算法，根据dir.y来计算天空的颜色。核心在getSeaColor这个函数上(**注意看如何mix反射的天空颜色和水体自身的颜色**)

{% highlight c linenos %}
vec3 getSeaColor(vec3 p, vec3 n, vec3 l, vec3 eye, vec3 dist) {  
    //菲涅尔系数计算
    float fresnel = clamp(1.0 - dot(n,-eye), 0.0, 1.0);
    fresnel = pow(fresnel,3.0) * 0.65;
    vec3 reflected = getSkyColor(reflect(eye,n));    //反射天空色
    vec3 refracted = SEA_BASE + diffuse(n,l,80.0) * SEA_WATER_COLOR * 0.12;  //水体颜色
    vec3 color = mix(refracted,reflected,fresnel);//fresnel越大，反射色的比例越大
    float atten = max(1.0 - dot(dist,dist) * 0.001, 0.0);
    color += SEA_WATER_COLOR * (p.y - SEA_HEIGHT) * 0.18 * atten;
    color += vec3(specular(n,l,eye,60.0));
    return color;
}
{% endhighlight %}

mainImage的最后是gamma矫正，至此代码分析完毕。

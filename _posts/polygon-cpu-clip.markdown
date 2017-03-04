---
layout: post
category: "dev"
title: "关于任意凸多边形的CPU剪裁问题"
tags: "Graphics"
summary: "凸多边形的剪裁"
---

# 综述

在图像渲染的时候，我们可以使用硬件剪裁（在OpenGL中也就是glScissor或者使用stencilbuffer)。但这样做的劣势是必须增加一个drawcall，于是就涉及到了比较大的切换开销问题（具体的可以参考这篇[知乎帖子](https://www.zhihu.com/question/27933010)。

要在程序中做任意多边形的剪裁，也就需要把多边形和剪裁框的交算出来，然后在分解成triangle fan给硬件进行渲染。这其中涉及到的计算量显然要比单纯的AABB求交来的“重“的多。主要涉及三个方面：

* 线段求交点
* 如何判断点在凸多边形内部
* 如何求凸包(convex hull)

所有的计算都基于向量积，但具体的实现有一些坑需要填（详细的可以参考下面的代码）。

# 算法

1. 求多边形P和剪裁框C的所有交点的集合S
2. 把P的所有在C内部的顶点加到集合S中
3. 把C的所有在P内部的定点加到集合S中
4. 对S进行去重操作
5. 选取S的一个顶点，将其余顶点按照逆时针/顺时针排序然后顺次构建TriangleFan

对于这个算法的简单证明，如果点p在交集的边界上，则显然p或者在P的边界上，或者在C的边界上，但不是both。那么这个边界的顶点只有两种可能，要不然是P和C的边界相交而成，或者是P和C自己的边相交而成。前者由算法第一步求出，后者由算法2，3步补充。

# DEMO

<script type="text/javascript">
			var EPSILON = 0.000001;
			var ctx;
			var canvas;
			var dragPos;
			var dragging = false;
			
			var clip_y_min = 200,
			    clip_y_max = 400,
			    clip_x_min = 200,
			    clip_x_max = 600;
			var polygon;
			var temp_points =[];
		
			function Point(x,y) {
				this.x = x;
				this.y = y;
			}
			
			Point.prototype.draw = function() {
				ctx.beginPath();
				ctx.arc(this.x, this.y, 4, 0, Math.PI*2, true);
				ctx.closePath();
				ctx.fillStyle = '#ff0000';
				ctx.fill();
			}
			Vector2 = Point;
			Vector2.prototype.cross = function(v) {
				return this.x * v.y - this.y * v.x;
			}
			Vector2.prototype.dot = function(v) {
				return this.x * v.x + this.y * v.y;
			}

			Vector2.prototype.sqrLen = function() {
				return this.x * this.x + this.y * this.y;
			}
			
			function vec2p(p1, p2) { 
				return new Vector2(p2.x - p1.x, p2.y - p1.y); //p1->p2
			}

			function intersect_segments(p1, p2, p3, p4) { //求p1p2和p3p4的交点
				var points = []
				var r = vec2p(p1, p2);
				var s = vec2p(p3, p4);
				var t = vec2p(p1, p3);

				var rs = r.cross(s);
				var tr = t.cross(r);
				if (Math.abs(rs) < EPSILON) {
					if (Math.abs(tr) < EPSILON) {
						//共线
						var t0 = t.dot(r) / r.dot(r);
						var t1 = vec2p(p1, p4).dot(r) / r.dot(r);
						var tmax, tmin;
						if (t0 > t1) {
							tmax = t0;
							tmin = t1;
						} else {
							tmax = t1;
							tmin = t0;
						}
						if (tmax < 0 || tmin > 1) {
							//共线不相交
						} else { //共线两个交点
							if (tmin < 0) tmin = 0;
							if (tmax > 1) tmax = 1;
							points.push(new Point(p1.x+tmin*r.x, p1.y+tmin*r.y));
							points.push(new Point(p1.x+tmax*r.x, p1.y+tmax*r.y));
						}
					} else {
						//平行不相交
					}
				} else {
					//相交
					var u = tr / rs, v = t.cross(s) / rs;
					if (0 <= u && u <= 1 && 0 <= v && v <= 1) {
						points.push(new Point(p3.x+u*s.x, p3.y+u*s.y));
					}
				}
				return points;
			}
			
			function Polygon(points) {
				this.points = points;
			}
			
			Polygon.prototype = {
				clip:　function(xmin, xmax, ymin, ymax) {
					var p_clip_flags = new Array(this.points.length);
					var allin = true;
					this.points.forEach(function(p, idx){
						if (xmin <= p.x && p.x <= xmax && ymin <= p.y && p.y <= ymax) {
							p_clip_flags[idx] = true;
						}
						else {
						    p_clip_flags[idx] = false;
						    allin = false;							
						}
 					});
					if (allin) return this;  //如果都在剪裁框内，直接返回

					var c_flags = new Array(4);
					var clipLT = new Point(xmin, ymin), clipRT = new Point(xmax, ymin),
					    clipRB = new Point(xmax, ymax), clipLB = new Point(xmin, ymax);
					c_flags[0] = this.hitTest(clipLT);
					c_flags[1] = this.hitTest(clipRT);
					c_flags[2] = this.hitTest(clipRB);
					c_flags[3] = this.hitTest(clipLB);
					if (c_flags[0] && c_flags[1] && c_flags[2] && c_flags[3])
						return new Polygon([clipLT, clipRT, clipRB, clipLB]); //如果剪裁框完全落在多边形内则返回剪裁框

					//求所有交点以及在两个多边形内的顶点的集合，之后进行一次闭包运算即可
					var convex = [];
					for (var i = 0;i < this.points.length; ++i)
						if (p_clip_flags[i])
							convex.push(this.points[i]);
					if (c_flags[0]) convex.push(clipLT);
					if (c_flags[1]) convex.push(clipRT);
					if (c_flags[2]) convex.push(clipRB);
					if (c_flags[3]) convex.push(clipLB);
					for (var i = 0;i < this.points.length; ++i) {
						var next = (i == this.points.length-1) ? 0:(i+1);
						convex = convex.concat(intersect_segments(this.points[i], this.points[next], clipLT, clipRT));
						convex = convex.concat(intersect_segments(this.points[i], this.points[next], clipRT, clipRB));
						convex = convex.concat(intersect_segments(this.points[i], this.points[next], clipRB, clipLB));
						convex = convex.concat(intersect_segments(this.points[i], this.points[next], clipLB, clipLT));
					}
					//TODO: POINT去重
					var _convex = [];
					convex.forEach(function(p) {
						var notexists = true;
						for (var i = 0;i < _convex.length; ++i) {
							if (vec2p(_convex[i], p).sqrLen() <= EPSILON) {
								notexists = false;
								break;
							}
						}
						if (notexists) _convex.push(p);
					});
					convex = _convex;
					if (convex.length < 3) return null;
					var origin = convex.shift();
					convex.sort(function(p1, p2) {
						var t = vec2p(origin, p1).cross(vec2p(origin, p2));
						if (t < 0) return -1;
						else if (t == 0) return 0;
						else return 1;
					});
					convex.unshift(origin);
					return new Polygon(convex);
				},
				hitTest: function(pt) {
					var ge0 = true, le0 = true;
					for (var i = 0;i < this.points.length; ++i)
					{
						var next = i == this.points.length - 1 ? 0: i + 1;
						var c = vec2p(this.points[i], pt).cross(vec2p(this.points[i], this.points[next]));
						if (c < 0) ge0 = false;
						if (c > 0) le0 = false;
					}
					return ge0 || le0;
				},
				move: function(dx, dy) {
					this.points.forEach(function(p){
						p.x += dx;
						p.y += dy;
					});
				},
				draw: function(fillColor) {
					ctx.beginPath();
					this.points.forEach(function(p,idx){
						if (idx == 0)
							ctx.moveTo(p.x, p.y);
						else
							ctx.lineTo(p.x, p.y);
					});
					ctx.closePath();
					ctx.fillStyle = fillColor;
					ctx.fill();
				}
			}
			
			function getCanvasCord(x,y) {
				return new Point(x - canvas.offsetLeft, y - canvas.offsetTop);
			}
			
			function init()
			{
				canvas = document.getElementById('demo_canvas');
				ctx = canvas.getContext('2d');
			
				setInterval(draw, 20);
			
				canvas.addEventListener('click', function(e) {
					if (!polygon)
					{
						temp_points.push(getCanvasCord(e.clientX, e.clientY));
						if (temp_points.length == 4) {
							polygon = new Polygon(temp_points);
							temp_points = [];
						}
					}
				});
			
				canvas.addEventListener('mousedown', function(e){
					var pt = getCanvasCord(e.clientX, e.clientY);
					if (polygon && polygon.hitTest(pt)){
						dragPos = pt;
						dragging = true;
					}
				});
				canvas.addEventListener('mousemove', function(e) {
					if (dragging) {
						var newPos = getCanvasCord(e.clientX, e.clientY);
						polygon.move(newPos.x - dragPos.x, newPos.y - dragPos.y);
						dragPos = newPos;
					}
				});
			
				canvas.addEventListener('dblclick', function(e){
					if (polygon) polygon = null;
				});
			
				canvas.addEventListener('mouseup', function(e){
					if (dragging) dragging = false;
				});
			}
			
			function draw()
			{
				ctx.clearRect(0,0,800,600);
			
				ctx.strokeStyle="#000000";
				ctx.strokeRect(clip_x_min, clip_y_min, 
					clip_x_max-clip_x_min, clip_y_max-clip_y_min);
			
				if (polygon) {
					polygon.draw('#ff0000');
					
					var p = polygon.clip(clip_x_min, clip_x_max, clip_y_min, clip_y_max);
					if(p) p.draw('#0000ff');
				} else if (temp_points.length > 0) {
					temp_points.forEach(function(p){
						p.draw();
					});
				}
			}
</script>

<style type="text/css">
#demo_canvas {
    border: 1px solid red;
}
</style>

<p>按顺序点击canvas区域创建一个凸四边形，双击取消当前四边形</p>
<canvas id="demo_canvas" width="800" height="600" onload="init();">浏览器不支持html5 canvas!</canvas>


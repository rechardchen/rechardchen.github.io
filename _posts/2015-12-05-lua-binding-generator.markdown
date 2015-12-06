---
layout: post
category: "dev"
title: "天天酷跑集成Lua的过程"
tags: "Lua"
summary: "天天酷跑集成Lua的历程"
---

## 背景

天天酷跑基于自研引擎NPEngine，在游戏运营过程中暴露出修改bug过程冗长（尤其是iOS平台），项目编译耗时长（万恶的C\+\+）等问题。经过短暂的技术调研，项目组决定集成Lua脚本引擎。

然而作为一个纯C\+\+实现的引擎，NP在设计之初就没有考虑到脚本化的需求。大量的原生接口需要导出到脚本环境，这个工作如果采用人力，过程将十分耗时且容易出错。在考察了Cocos2dx的脚本绑定技术之后，我们决定采用自动生成绑定代码的方式来完成这个工作。

## 工具

1. [binding-generater](https://github.com/cocos2d/bindings-generator)
  
	为了能在cocos2dx中使用Javascript和Lua开发的接口绑定代码生成工具。基于Python2.7，py-yaml（Python yaml模块），cheetah（类似Jinja2，模板引擎）和libclang（LLVM前端Clang的Python绑定）。
    
2. [tolua\++](http://sourceforge.net/projects/toluapp.berlios/)
	
    tolua\++扩展了tolua，用于将C/C++代码导入到lua脚本执行环境。**tolua\+\+加入了对C\+\+模板的支持**。
    
按照tolua的文档，可以直接用于接口的导出，只是我们还需要写一些（da liang）配置文件，这就是为什么我们需要binding-generator的理由。

使用Python脚本用Clang将代码中我们需要的接口提取出来自动生成tolua的接口文件不就行了？This is a good idea，然而binding-generator走的更远。事实上它只用了tolua\+\+提供了的基础函数，而绑定代码都由自己生成（这里要再由衷的赞叹一下Lua的capi的简洁设计！）。

## binding-generator的使用

秉持光荣的拿来主义传统，我们直接把cocos2dx源码tools文件夹下的`bindings-generator`和`tolua`两个文件夹拷贝到我们自己的引擎下面。然后，

1. 设置环境变量NDK\_ROOT为自己机器上Android-ndk的路径
2. 修改`tolua/genbindings.py`的内容，把cocosdir参数换成我们自己的头文件路径（这里定义的配置都可以在步骤3中的ini文件中直接引用）。把`main`中的`cmd_args`字典的内容清空（这些都是cocos2dx的导出配置）
3. 删除`tolua`下所有`xxx.ini`文件，仿照这些文件的格式写自己引擎的配置文件，其格式大致如下
	* 每个section都会被转化成一个单独的接口hpp&cpp文件，其中prefix参数是导出的函数的前缀用来避免名字冲突，当然也可以通过配置target\_namespace参数来达到目的
	* headers参数定义了Clang解析的头文件，也就是要导出的类所在的头文件，用空格隔开
	* classes参数定义了要导出的类的列表，两两用空格隔开。这里的类名可以用正则表达式（使用的时候在头尾加上^和$）
	* 不是要导出的类的每个public方法对我们来说都是需要的，skip参数定义了哪些方法不用导出
	* 一般来说我们要用的数据结构都不是在lua中new出来的，所以对于要导出的类我们不需要构造函数，在abstract\_classes里配置不导出构造函数的类列表（如果一个类被导出了，但是没有被标记为abstract\_class，则在lua中会相应的生成一个C:new()构造函数)
4. 关键的一步，修改`bindings-generator/targets/lua/conversions.yaml`，这里主要需要配置lua和c\+\+参数之间的互相转换函数。to\_native定义了lua到c\+\+，而from\_native则定义了c\+\+到lua的转换，这里定义的字符串会被用来替换templates下的模板中相应的转换调用

	举个from\_native的例子，我们在C\+\+中有一个结构`PlayerSubTaskInfo`，现在需要在lua中用到这个结构，首先在实现文件中定义转换函数
    
    ```Cpp
    void luaval_by_native(lua_State* L, const PlayerSubTaskInfo& info)
    {
        if (NULL == L) return;
        lua_newtable(L);
        lua_pushinteger(L, info.dwSubTaskId);
        lua_setfield(L, -2, "subTaskId");
    }
    ```
    然后需要做的就是在from\_native里面加一行转换规则`"PlayerSubTaskInfo": "luaval_by_native(L, ${in_value})"`。这样之后我们在lua中就可以以一个table的形式来访问c\+\+传过来的PlayerSubTaskInfo对象，这个表的subTaskId域即是原对象的dwSubTaskId。
    
    **与ini文件中定义要导出的类对象不同，from\_native中定义的对象在导出后都是以table的形式在lua中使用，我们只是从中取数据（以field的形式）。如果需要修改数据or调用方法，那么必须在ini文件中定义。数据类型的转换函数是luaval_by_native，而对象类型则是luaval_by_object，区分这一点很重要，使用过程中发现这是常见的错误之一。**
    
在使用binding-generator的过程中，一旦导出错误都会在控制台输出错误原因，这方便了配置错误的定位。


## tolua\+\+做了什么

至此，我们用binding-generator生成了绑定代码，然后我们将这些绑定文件添加到工程中去。编译，链接，OK，happy ending！我们的故事似乎可以结束了！等等，还有一个重要角色尚未登场！作为重要的组成部分，tolua\+\+究竟做了什么？

事实上，在binding-generator导出的大量代码中，包括了对tolua\+\+的调用。以NPEngine中的NPNode类为例，绑定该类的方法如下

```Cpp
int NPLuaRegisterCommon_NPNode(lua_State* L)
{
	tolua_usertype(L, "NPNode");
	tolua_cclass(L, "NPNode", "NPNode", "NPRenderObject", NULL);
	
    tolua_beginmodule(L, "NPNode");
    	tolua_function(L, "AddChild", NPNode_AddChild);
        ...
    tolua_endmodule(L);
    
    return 1;
}
```

这段代码将NPNode以metatable的形式注册到Lua中，而这个元表中包含的正是我们导出的函数（如AddChild）。当我们在Lua中接受一个C\+\+的NPNode对象的时候，运行的是luaval_by_object这个函数，

```Cpp
void tolua_pushusertype_internal (lua_State* L, void* value, const char* type, int addToRoot)
{
	...
    luaL_getmetatable(L, type);
    if (lua_isnil(L, -1)) {
    	lua_pop(L, 1);
        return;
    }
    lua_pushstring(L, "tolua_ubox");
    lua_rawget(L, -2);
    if (lua_isnil(L, -1)) {
    	lua_pop(L, 1);
        lua_pushstring(L, "tolua_ubox");
        lua_rawget(L, LUA_REGISTERINDEX);
    }
    lua_pushlightuserdata(L, value);
    lua_rawget(L, -2);
    if (lua_isnil(L, -1)) {
    	lua_pop(L, 1);	/*stack: mt ubox*/
        lua_pushlightuserdata(L, value);
        *(void**)lua_newuserdata(L, sizeof(void*)) = value;	/*stack: mt ubox value newud*/
        lua_pushvalue(L, -1);	/*stack: mt ubox value newud newud*/
        lua_insert(L, -4); 	/*stack: mt newud ubox value newud*/
        lua_rawset(L, -3);	/*stack: mt newud ubox*/
        lua_pop(L, 1);	/*stack: mt newud*/
        lua_pushvalue(L, -2);	/*stack: mt newud mt*/
        lua_setmetatable(L, -2);	/*stack: mt newud*/
#ifdef LUA_VERSION_NUM
		lua_pushvalue(L, TOLUA_NOPEER);
        lua_setfenv(L, -2);
#endif
    } else {
    	...
    }
    ...
}
```

这里value是NPNode对象的指针，因为light userdata是不能有元表的，所以这里绕了一个弯子，对于每个指针新建了一个userdata对象，其内存值就是指针的值。代码的最后把NPNode对应的元表设置为这个userdata的元表，且将这个userdata作为返回值返回给脚本。在这之后，如果我们在脚本中调用该对象的AddChild方法，则事实上实际上调用的是NPNode元表里面的AddChild，这个逻辑藏在`tolua_usertype(L, "NPNode")`中，

```Cpp
TOLUA_API void tolua_classevents (lua_State* L)
{
    lua_pushstring(L,"__index");
    lua_pushcfunction(L,class_index_event);
    lua_rawset(L,-3);
    ...
}
```

NPNode元表的`__index`域被设置为函数`class_index_event`。通过查看代码可以看出倘若在Lua中去调用NPNode对象的AddChild方法，最终得到调用的是NPNode元表中对应的NPNode\_AddChild。

至此，整个绑定的主体部分豁然开朗！

## 总结和拓展

* 通过查看tolua++的代码，可以对Lua语言的各种细节之处作深入的了解，比如元表，注册表，环境，垃圾收集等等，内容之多可以单独作为一篇文章来讲。
* Binding-generator使用到的libclang使用了Python作为脚本，这为我们扫描项目源代码，借clang来做一些代码静态分析成为可能。
* tolua++提供了对c\+\+模板的支持，这个特性我们尚且没有用到，然而在酷跑项目中模板的使用也不新鲜，所以加入对c\+\+模板的支持可以加入下一步的计划。


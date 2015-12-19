---
layout: post
category: "dev"
title: "C++全局构造小记"
tags: "C++"
summary: "ComponentManager的注册优化"
---

码农的生活总是过的很快，班加着加着就来到了年末，可耻的奔三去了，哎。

这周工作上的事情不是很多，所以能静下心来把项目中的问题梳理一下，该重构的重构，该重写的重写，该优化的优化。期间遇到一个值得玩味的问题，至今尚未完全解决。

### 一个组件的工厂类

问题来自于游戏的Entity-Component系统，这里将其简化和抽象的表述出来。游戏里的每个实体包含了若干个组件，为了达到组件最大程度的复用，我们采取配置文件的形式。一个典型的配置可能如下所示，

```Json
{
    "Components": [
        {
            "ClassName": "AIComponent",
            "Param1": "param1"
        },
        {
            "ClassName": "TransformComponent",
            "Param2": "param2"
        }
    ],
    "EntityName": "EntA"
}
```

这段Json代码定义了一种包含了两个组件AIComponent和TransformComponent的Enitity类型EntA。要实现这样的配置驱动，我们需要一个组件的工厂类ComponentManager，它能根据组件的类名创建出各个不同的组件实例出来，

```Cpp
class ComponentManager {
public:
	Component* CreateComponent(const std::string& className);
};
```

在Component基类里，定义Creator函数类型，以及需要的宏

```Cpp
class Component {
virtual ~Component() {}
};

typedef Component* ComponentCreator();

#define DECLARE_COMP \
	public: static Component* Create()
#define IMPLEMENT_COMP(comp) \
	Component* comp::Create() { \
    	return new comp();	\
    }
```

对于子类组件AIComponent

```Cpp
class AIComponent: public Component {
	DECLARE_COMP;
};
```
```Cpp
IMPLEMENT_COMP(AIComponent);
```

改造ComponentManager使之能够根据组件的名称来创建组件，以及注册不同的组件跟Creator的映射关系，

```Cpp
class ComponentManager {	//注: ComponentManager是一个单件
	...
public:
	Component* Create(const std::string& className) {
    	std::map<std::string, ComponentCreator*>::iterator it = m_Creators.find(className);
        if (it != m_Creators.end()) {
        	return (*(it->second))();
        } else {
        	return NULL;
        }
    }
    void RegisterComponent(const std::string& className, ComponentCreator* creater) {
    	m_Creators[className] = creater;
    }
private:
	std::map<std::string, ComponentCreator*> m_Creators;
};
#define REGISTER_COMPONENT(comp) \
	ComponentManager::Instance()->RegisterComponent(#comp, &(comp::Create))
```

然后我们所需要做的就是在游戏开始的时候把所有的组件注册到ComponentManager中去，

```Cpp
void InitGame() {
	...
	REGISTER_COMPONENT(AIComponent);
	REGISTER_COMPONENT(TransformComponent);
	...
}
```

### 自动注册组件方案

上述方法的问题在于，每次写一个新的组件，必须要在初始化函数里加一行`REGISTER_COMPONENT(NewComponent)`，且这个调用需要包含对应的组件的头文件，代码非常不好看，且增加了依赖关系。遂想到用C\+\+的全局构造来帮助自动注册组件。

改写Component.h

```Cpp
class Component {
virtual ~Component() {}
};

typedef Component* ComponentCreator();
class ComponentRegister {
public:
	ComponentRegister(const char* className, ComponentCreator* creator) {
    	ComponentManager::Instance()->RegisterComponent(className, creator);
    }
}

#define DECLARE_COMP \
	static Component* Create()
#define IMPLEMENT_COMP(comp)\
	Component* Create() {\
    	return new Comp();\
    }\
	static const ComponentRegister comp##RegisterVar (#comp, &(comp::Create));
```

对于组件AIComponent，在入口函数被调用之前一个名为AIComponentRegisterVar的静态常量将被构造，在它的构造函数里面我们把AIComponent注册到ComponentManager里面去。

看起来已经大功告成了！现在我们无需再手动的一个一个的为组件注册了！世界又恢复了安宁！

### 问题

上面的方案看似天衣无缝，然而在vs2010里，实测下来并不是所有的ComponentRegister对象都能在入口函数被调用之前得到初始化。也就是说如果在main调用中要创建一个新的组件，可能得到的是一个空指针，因为此时这种类型还没有被注册！:(

实在没辙了！最后终于在[Stackoverflow](http://www.stackoverflow.com)大神的指导下，查阅了C\+\+11的语言标准文档[ISO/IEC CD 14882](http://webstore.ansi.org/RecordDetail.aspx?sku=INCITS%2FISO%2FIEC+14882-2012)（当然我没有买，但是如果你无比热爱C\+\+可以选择花30刀支持一下，O(∩_∩)O~），在3.6.2节关于non-local变量的初始化中有这样一段

> It is implementation-defined whether the dynamic initialization of a non-local variable with static storage duration is done before the first statement of main. If the initialization is deferred to some point in time after the first statement of main, it shall occur before the first odr-use of any function or variable defined in the same translation unit as the variable to be initialized.

也就是说，标准里只规定了对于那些需要动态初始化的static存储的变量，他的动态初始化会在同一个编译单元的函数或者变量被第一次使用之前获得调用。所以说，如果希望ComponentRegister对象在main之前被调用，标准唯一保证的做法是把它写在main函数定义的cpp文件中。

至此宣告了尝试的失败，要实现自动注册只能想其他办法啦。期待后续解决方案，这次就写到这里。




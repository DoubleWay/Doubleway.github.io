---
layout:     post
title:      JetPack MVVM
subtitle:   
date:       2020-06-15
author:     DoubleWay
header-img: img/post-bg-ios9-web.jpg
catalog: 	 true
tags:
    - Android
    - MVVM
---

## JetPack MVVM

google为了帮助开发者更好的，更规范的进行开发，将各种能够帮助开发的套件，组件，整合到了一起。这就是JetPack，所以JetPack里面不仅包含最新的的东西，只要是对开发有帮助的库都在里面。

MVVM就是基于JetPack库进行开发的一套架构，相当于MVC的进化版。因为Android的开发比较特殊，在activity或fragment里面，既要处理部分视图关系，也要进行业务逻辑。所以像MVC那种，view和controller分开，视图和业务逻辑分开处理的开发模式，就不大适合Android开发。因为activity既像view，又像controller，逻辑就可能比较混乱。

这时候可能会说，不是还有MVP么，MVP相当于MVC的演化版，从模式上来看和MVC非常相似，但是相对与MVC来说MVP的view和model解耦就比较彻底，具体的实现逻辑都交给prenester来做，在view中，调用相关接口就行。在MVP中View并不直接使用Model，它们之间的通信是通过Presenter (MVC中的Controller)来进行的，所有的交互都发生在Presenter内部，而在MVC中View会直接从Model中读取数据而不是通过 Controller。但是，因为将渲染界面都交给prenester来做，view和prenester的交互就会过于频繁，当页面和逻辑变得越来越复杂的时候，prenester就会变得越来越庞大臃肿，过多的接口也不便于维护。

而面临这种情况，MVVM就是应运而生，面临UI越来复杂，接口变多的情况下，MVVM的双向绑定机制，UI和数据改动一方，另一方也能够及时更新。

M：model,从各种数据源中获取数据

V：view，Android里面主要指定就是activity和fragment，负责和用户进行交互，以及驱动viewmodel从model中获取数据

VM：viewmodel，用于将view和model进行关联，在view中通过vewimodel获得数据后，利用他的数据绑定机制，将数据刷新到UI上

现在就来看看在MVVM中主要用到的的几个套件：

### JetPack LifeCycle

在LifeCycle之前，生命周期基本纯靠手工来进行管理，在代码的管理和一致性上就可能会有大量的问题，比如一些组件要在每个依赖的它的onResume和onPause里面去激活，绑定，终止等，这样稍微疏忽一点就会出现问题，而且以后要增加一些方法，比如状态监听之类，分散在各个activity里面的代码也不便于修改。

而LifeCycle将对生命周期管理的复杂操作，全部封装起来，而开发者只需要在作为LifeCylceOwener的基类中getLifeCycle.addObserves()，就可以在第三方内部中监听到生命周期的变化

### JetPack LiveData

LiveDate是一个能够感知Activity和Fragment生命周期的数据容器（数据持有类），当LiveData持有的数据发生改变时，会通知相关绑定的界面更新数据。LiveData还持有界面的LifeCycle，当界面（LifeCycleOwner）的生命周期处于Started和Resume状态，才会去通知。当LifeCycleOwner被销毁的时候LiveData也会被销毁。所有不用担心内存泄露

### JetPack ViewModel

ViewModel 将视图的数据和逻辑从具有生命周期特性的实体（如 Activity 和 Fragment）中剥离开来。直到关联的 Activity 或 Fragment 完全销毁时，ViewModel 才会随之消失，也就是说，即使在旋转屏幕导致 Fragment 被重新创建等事件中，视图数据依旧会被保留。ViewModels 不仅消除了常见的生命周期问题，而且可以帮助构建更为模块化、更方便测试的用户界面。

### JetPack DataBInding

DataBinding，顾名思义就是数据绑定，就是将数据和UI界面中的组件进行绑定，分为单项绑定和双向绑定。单向绑定就是数据的变化会驱动UI布局的变化，而双向绑定还包括UI界面上数据的变化会是绑定的应用数据而发生变化。
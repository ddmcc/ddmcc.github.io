---
layout: post
title:  "React组件生命周期"
date:   2019-09-02 23:25:18
categories: 前端
tags:  react
toc: true
---


 　　React组件的一生主要分为四个阶段分别为初始化（`Initialization`），挂载（`Mounting`），更新（`Updation`），卸载（`Unmounting`），
 下面介绍挂载和更新的一些方法。


<!-- more -->

## 16.3之前
 
![markdown](https://ddmcc-1255635056.file.myqcloud.com/f9d08583-fb5b-413f-82b0-9bd462cff551.png)
 
 ---
#### 挂载阶段

 　　挂载是指组件渲染并构造dom元素，然后将元素插入到页面的过程， **这是一个从无到有的过程** 。在这个过程中主要有以下几个方法：
 
 
 - componentWillMount
 
它会在组件渲染之前执行，在此方法之后，组件将渲染挂载，在组件的整个生命周期中，这个方法只会被执行一次，并且是服务端渲染唯一会调用的生命周期函数。~~所以可以在此方法中做一些初始化的工作。~~


- render

将组件渲染到页面中。它是一个`纯方法`，输出由输入决定


- componentDidMount

此方法在生命周期中也只会执行一次，并且在组件挂载之后。在此方法中，可以获取到页面的dom节点。子组件的方法会在父组件之前调用


#### 更新阶段

 　　更新阶段是指调用setState或props改变而触发，或者调用`forceUpdate`来重新渲染组件并把组建的变化显示到DOM元素的过程。
 **这是一个变化的过程**
 
 - **1,当通过调用setState更新状态触发**
 
    - shouldComponentUpdate 
    
       此方法在收到新的props或者状态更新的过程中会被调用。见名知意，此方法可以控制组件是否要被重新渲染。方法返回true或false，当返回false时，组件不会被渲染。默认返回为true
    
     - componentWillUpdate
     
       此方法将在`shouldComponentUpdate`返回为true后被调用，类似`componentWillMount`，只不过它是在每次被重新渲染的时候都会被调用一次
        
    - render
    
       同上
       
    - componentDidUpdate
    
      当组件被更新到DOM之后会被触发
      
      
- **2,父组件传入新的props**

    - componentWillReceiveProps(nextProps)
    
      当组件收到新的props会被触发，在首次渲染不会被调用。可以判断是否要根据父类传入新的props而要更改state，在此方法中调用setState不会重复渲染

    - shouldComponentUpdate
    
      同上
      
    - componentWillUpdate
      
      同上
      
    - render
    
      同上
      
    - componentDidUpdate
    
      同上
      
      
- **3,调用forceUpdate**

 　　当不想通过state或props进行改变触发重新渲染时，可以调用forceUpdate，此操作会跳过该组件的 `shouldComponentUpdate`。但其子组件会触发正常的生命周期方法，包括`shouldComponentUpdate`方法。
通常不推荐使用该方法。


## 16.3版本

 　　在16.3中新增`static getDerivedStateFromProps(nextProps, prevState)`和`getSnapshotBeforeUpdate(prevProps, prevState)`，同时废弃了`componentWillMount`，`componentWillUpdate`，`componentWillReceiveProps`三个方法。
 先说为什么要废弃这三个在渲染之前的方法
 
 
 　　在React v16中发布了React Fiber，简单说一下Fiber
 
 > 在现有React中，更新过程是同步的，这可能会导致性能问题。在更新的过程中要做很多事情，比如调用各个组件的生命周期函数，计算和比对Virtual DOM，最后更新DOM树，这整个过程是同步进行的。
 >
 >Fiber就是把所有任务分成一片一片，每当执行完一片，就会给其他任务执行的机会（感觉可以看成线程竞争cpu时间片😄），所以有可能当一个更新任务还没完成，就被其它优先级高的任务给中断了。
 >重点是被中断的任务不是在中断的地方接着执行，它是重新开始的...重新开始...
 >
 >Fiber分为两个阶段，第一个阶段允许被打断，而第二阶段则不允许。以render为界，可以被打断的生命周期函数就有
 >
>- componentWillMount
>- componentWillReceiveProps
>- shouldComponentUpdate
>- componentWillUpdate
>- render

 　　**所以如果在`componentWillMount`和`componentWillUpdate`做一些后台数据的获取或者定时器的初始化等的工作，将不再适合，因为这些方法有可能会被执行多次**。官方为了不让大家在上面的方法中继续做不正确的事情，
 干脆把这几个方法给废弃了，并推出了新的方法`getDerivedStateFromProps`，如果之前在废弃的方法中的实现比较纯，那么直接移到新的方法中即可。它是一个静态方法，不能使用this，结果只由输入去改变，**返回的结果直接传给setState**。
 **如果是要去后台获取数据或者初始化的工作则可以移到`componentWillMount`和`componentWillUpdate`方法中**。这两个方法不会有被执行多次的可能。
 
 ---

![markdown](https://ddmcc-1255635056.file.myqcloud.com/ab61714e-cb0e-429b-82ed-e5f03eff57f9.png)

 ---
 图来源于[http://projects.wojtekmaj.pl/react-lifecycle-methods-diagram/](http://projects.wojtekmaj.pl/react-lifecycle-methods-diagram/)
 
#### 挂载阶段
 
- getDerivedStateFromProps
 
   `static getDerivedStateFromProps(nextProps, prevState)` 这是一个静态的方法，会传入新的props和原来的状态值。在调用 render 方法之前调用，它应返回一个对象来更新 state，如果返回 null 则不会更新内容。
   官方文档也说了这个方法适用于state的值在任何时候都取决于props，如果要做 **有副作用的操作则应使用DidMounting，DidUpdate或在prop更改时“重置”某些state则可以考虑使用受控的组件** 
   
   如图此方法也可能会被暂停，中断或重新开始，因为处于`Render 阶段`
   
- render
 
    方法也可能会被暂停，中断或重新开始
    
- componentDidMount
    
    同上
 
#### 更新阶段

- **1,调用setState更新状态触发**

    - shouldComponentUpdate
        
        调用`setState`触发的渲染并不会调用getDerivedStateFromProps，而是直接到了`shouldComponentUpdate`。**需要注意的是shouldComponentUpdate也有执行多次的可能**
     
    - render
    
        同挂载阶段，也可能多次执行

    - getSnapshotBeforeUpdate(prevProps, prevState)
     
        这也是在V16.3中新加入的方法，只有在更新阶段，`render`方法和真正DOM被改变之间执行，如图这时可以读取dom，此方法不会被中止或重新开始。**方法返回的值会在`componentDidUpdate`方法的第三个参数传入**
     
     >componentDidUpdate(prevProps, prevState, snapshot)
     
    - componentDidUpdate
    
       同上
     
 
- **2,父组件传入新的props**

    - getDerivedStateFromProps
    
        在更新阶段的`setState`方法而触发的重新渲染会被调用，解释如上。**需要注意的是，此方法也在“纯净不包含副作用”阶段，所以可能会被执行多次。
        
    - shouldComponentUpdate 
    
        同上
        
    - render
    
        同上
        
    - componentDidUpdate
    
        同上
        
- **3,调用forceUpdate**

    此过程和16.3之前一样会直接到render方法
    
    
## ^16.4

 　　在16.3中新加入的`getDerivedStateFromProps`方法只有在挂载和由`setState`引起的Updating才会被执行，这可能会给开发人员带来困扰，父组件传入新的props和forceUpdate并不会Updating。
 
 　　**在16.4中，React官方改正了这一点，无论是挂在阶段或是任何动作引起的Updating都会触发`getDerivedStateFromProps`**。
 
 　　所以16.4的生命周期方法如下图：
 
    

![markdown](https://ddmcc-1255635056.file.myqcloud.com/3b8e2d83-7d94-4461-b053-c8a71435bb65.png)
---
 　　解释：略
      

## 总结

 　　用静态函数getDerivedStateFromProps来取代被deprecate的几个生命周期函数，就是强制开发者在render之前只做无副作用的操作，而且能做的操作局限在根据props和state决定新的state，因为在render阶段生命周期方法可能会被执行多次。**废弃的方法会在17版本中删除**
 
 
## 参考

[https://zh-hans.reactjs.org/docs/react-component.html](https://zh-hans.reactjs.org/docs/react-component.html)

[React v16.3之后的组件生命周期函数](https://zhuanlan.zhihu.com/p/38030418)

[React Fiber是什么](https://zhuanlan.zhihu.com/p/26027085)

[http://projects.wojtekmaj.pl/react-lifecycle-methods-diagram/](http://projects.wojtekmaj.pl/react-lifecycle-methods-diagram/)
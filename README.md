# react-mobx-code-structure
使用mobx作为react的数据流虽然写起来更加随意,但不免会让代码随着项目的庞大而凌乱,在做了几个react+mobx的项目后开始总结和反思,如何才能让流程清晰,扩展容易
代码有足够的解构呢?


在最初没有任何设计的样子是这样的:

![最初没有设计的样子](https://github.com/daiyunchao/react-mobx-code-structure/blob/master/v1.jpg)

说明:

页面A 负责页面的整体呈现,其中包含很多相关的组件及子组件

ModelA ModelB ModelC 是独立的Model,字段基本上和服务器端定义的model差不多(这样定义是出于Model的复用)

页面A中有ModelA ModelB 的引用
在页面A中使用Model的伪代码如下:
```
import ModelA from './models/ModelA';
import ModelB from './models/ModelB';

class PageA extends React.Component{
  constructor(){
    super();
    
    //每次使用都是实例化一个model出来使用,为什么要实例化? model是公共的
    this.modelA= new ModelA();
    this.modelB= new ModelB();
  }
}
render(){
  //调用modelA的action,查询数据,因为Mobx是MVVM模式所以数据会自动绑定
  this.modelA.getModelADetail();
  this.modelB.getModelBDetail();
  
  return (
    <div>这里显示的是ModelA一个叫标题的玩意:{this.modelA.title}</div>
    <div>这里显示的是ModelA一个叫内容的玩意:{this.modelA.content}</div>
    <div>这里显示的是ModelB一个叫作者姓名的玩意:{this.modelB.userName}</div>
  )
}
```
通过上面的代码发现至少以下几个问题:
1. UI并不单纯,在PageA中做了很多业务逻辑的东西,如果业务很多的话,Page将会很庞大,而且有可能写具体业务的和做纯粹UI的是两个人,同时编辑相同文件,可能会diam冲突
2. Page间的数据传递和代码共享存在问题,因为在每一个页面使用时都使用公共的Model的实例(都是实例化出来的),所以数据无法共享,也无法传递数据到另外的页面
3. 如果在PageA中需要调用一个既包含ModelA的数据也包含ModelB数据的API,我的action是写在ModelA呢 还是ModelB呢? 好像都不太合适
4. 因为无法共享数据,可能会造成重复查询(当然这里也是可能,随便创建一个单例文件用于保存数据也是可以的)


针对上面这些问题 我做了以下的设计:

![重新设计后的结构](https://github.com/daiyunchao/react-mobx-code-structure/blob/master/v2.jpg)
说明&变化:

通过结构图可以看到
1. UI层基本没什么变化
2. 添加了页面级的控制文件,该文件是单例的(同一个项目中,理论上只有一个相同的page,所以pageControl文件我设计的是单例的),在该文件中包含该页面Model的实例引用和该页面的逻辑处理,实现了将逻辑和UI独立
3. 添加了pageControlBase,作为通用pageControl的父类,实现通用各个pageControl的通用方法
4. 添加了shareModels,共享模型数据,对于一个项目总有


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
4. 添加了shareModels,该文件是单例,共享模型数据,对于一个项目总有共有的模型(比如User),该文件用于存放那些共有的Model引用,方便页面之间的数据传递
5. 请求API的部分也有一些变化,原来请求API的只是发生在Model层面,为了解决上述当请求既包含ModelA的数据 也包含ModelB的数据时action应该放到哪的问题,现在这类的API可以放到PageControl中

新版本的的大致代码:

Page的伪代码:
```
import Model from './pageControl/PageAControl';
class PageA extends React.Component{
  constructor(){
    super();
    //因为是每一个PageControl是单例的,所以不用实例化,直接调用需要的action就行了
    Model.getPageADetail();
  }
}
render(){
  return (
    <!--在具体绑定数据时,所有的数据都来着PageControl的这个Model,PageControl会有modelA modelB 实例的引用-->
    <div>这里显示的是ModelA一个叫标题的玩意:{Model.modelA.title}</div>
    <div>这里显示的是ModelA一个叫内容的玩意:{Model.modelA.content}</div>
    <div>这里显示的是ModelB一个叫作者姓名的玩意:{Model.modelB.userName}</div>
  )
}
```


pageControl的伪代码:

```
import {observable, autorun, action, extendObservable, toJS} from "mobx";
import ShareModel from './ShareModel';
import ModelA from '../models/ModelA';
import ModelB from '../models/ModelB';
class PageAControl extends PageControlBase{

  //页面A的一些状态
  status={};
  
  constructor(){
    super();
    extendObservable(this.status,{
      isLogin:false//是否登录状态
    });
    
    //实例化model的,并且放入pageControl的引用中
    this.modelA= new ModelA();
    this.modelB= new ModelB();
    
    //如果modelB是一个全局的Model,比如User对象
    //将ModelB存放到ShareModel中去:
    ShareModel.add({"shareModelName":"modelB","shareModelEntity":this.modelB})
  }
  
  //获取页面详情:
  @action async getPageADetail(){
  
    //调用API,获取详情数据
    let pageADetail= await API.getPageADetail();
    
    //将详情页的数据存放到modelA中
    this.modelA.setModelInfo(pageADetail)
    
    //将详情页的数据存放到modelB中
    this.modelB.setModelInfo(pageADetail);
    
  }
}

//将PageControl构建成单例模式
export default new PageAControl();

```

PageControlBase的伪代码:
```

//pageControl的公告父类,用于实现各个pageControl的公共方法
class PageControlBase {
  pageControlCommonMethod(){
    //这是父类的一个通用方法
  }
}
export default PageControlBase;

```

ShareModel的伪代码:
```
class ShareModel{
  shareModels={};
  
  //添加共享model的方法
  add({shareModelName,shaeModelEntity}){
    //...
  }
  
  //获取共享model的方法
  get({shareModelName}){
    //...
  }
}
//单例
export default new ShareModel();
```

ModelA的伪代码:
```
import {observable, autorun, action, extendObservable, toJS} from "mobx";
class ModelA{
  data={};
  constructor(){
    @observable title=""
    @observable content=""
  }
  async getModelADetail(){
    //调用API,获取modelA的详情
    let modelADetail= await API.getModelADetail();
    this.title=modelADetail["title"];
    this.content=modelADetail["content"];
  }
  
  setModelADetail(data){
    this.title=data.title;
    this.content=data.content;
  }
}
export default ModelA
```
v2的文档结构:

```
--components(公共组件)
----userInfo.jsx
--page(页面)
----pageA.jsx
----pageB.jsx
--pageControl
----pageAControl.js
----pageBControl.js
--data
----shareModel.js
--models
----ModelA.js
----ModelB.js
```
代码复杂度是比没有设计的v1版本高一些,但UI和逻辑分离,页面间的数据传递(ShareModel)更方便了,逻辑也更清晰,我觉得还是值得的.

如果对我的代码结构有疑问或是有更好的解决方案可以提交:Issues一起讨论.


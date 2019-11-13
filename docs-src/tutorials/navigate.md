## 无限层级路由方案

### 背景
- 小程序原生页面存在层级限制，最多只能同时打开5层，超过5层时便会无法打开新页面（注：后来原生层级限制放宽至了10层，代码中已作更新，文档仍以5层为例）  

- 业务流程很容易一不小心就超过5层，如：首页-列表页-商品详情-下单页-订单详情页-私信页-...
- 访问回路很容易导致超过5层，如：首页-列表页-商品详情-查看更多-商品详情-查看更多-商品详情-...
- 为避免层级限制导致的无法打开问题和层级限制带来的交互路径限制，提出此路由方案

### 策略
- 修改小程序默认导航行为，自行维护完整历史记录
- 页面层级小于等于5时，导航行为与原生导航行为一致
- 请求打开第6层及以上时，逻辑层级记录完整历史，实际层级每次都是直接将第5层替换为目标页面
- 返回时，逻辑层级相应回退；若回退后逻辑层级大于等于5，则实际层级将第5层替换为回退后目标页面，否则实际层级回退到相应页面
- demo:

```txt
  逻辑层级 1 - 2 - 3 - 4 - 5
  实际层级 1 - 2 - 3 - 4 - 5
  
  打开
  
  逻辑层级 1 - 2 - 3 - 4 - 5 - 6
  实际层级 1 - 2 - 3 - 4 - 6
  
  打开，打开，打开
  
  逻辑层级 1 - 2 - 3 - 4 - 5 - 6 - 7 - 8 - 9 
  实际层级 1 - 2 - 3 - 4 - 9
  
  返回
  
  逻辑层级 1 - 2 - 3 - 4 - 5 - 6 - 7 - 8
  实际层级 1 - 2 - 3 - 4 - 8
  
  返回，返回，返回
  
  逻辑层级 1 - 2 - 3 - 4 - 5
  实际层级 1 - 2 - 3 - 4 - 5
  
  返回
  
  逻辑层级 1 - 2 - 3 - 4
  实际层级 1 - 2 - 3 - 4
```

### 补充策略：空白中转
- 从第6层及以上，如第9层返回时，理论上应直接将实际第5层直接替换为逻辑第8层页面，但由于系统返回行为无法取消，所以实际过程为：返回实际第4层-新开逻辑第8层
- 这一中转过程会使返回时实际第4层一闪而过，影响用户体验
- 为此，引入空白页中转：打开实际第5层时，将实际第4层替换为空白页；直到返回逻辑第4层时才将空白页替换回原始页面。
- demo:

```txt
  逻辑层级 1 - 2 - 3 - 4
  实际层级 1 - 2 - 3 - 4
  
  打开
  
  逻辑层级 1 - 2 - 3 - 4 - 5
  实际层级 1 - 2 - 3 - 空白页 - 5
  
  打开，打开，打开
  
  逻辑层级 1 - 2 - 3 - 4 - 5 - 6 - 7 - 8 - 9
  实际层级 1 - 2 - 3 - 空白页 - 9
  
  返回
  
  逻辑层级 1 - 2 - 3 - 4 - 5 - 6 - 7 - 8
  实际层级 1 - 2 - 3 - 空白页 - 8
  
  返回，返回，返回
  
  逻辑层级 1 - 2 - 3 - 4 - 5
  实际层级 1 - 2 - 3 - 空白页 - 5
  
  返回
  
  逻辑层级 1 - 2 - 3 - 4
  实际层级 1 - 2 - 3 - 4
```

### 附加功能：实例覆盖自动恢复
- 问题：wepy框架存在单实例问题，同一路径页面被打开两次时，其数据会相互影响，如：详情页A - 详情页B - 返回A，点击查看大图 - B的图片（而不是A的图片）
  详见issue：[两级页面为同一路由时，后者数据覆盖前者](https://github.com/Tencent/wepy/issues/322)
- 策略：返回时，若判断目标页面数据已被覆盖，则自动予以恢复
- 引入：可配，参见“安装”小节

### 附加功能： 免并发  
- 问题：用户连续快速点击多个/多次按钮时，会一次性打开多个窗口，一则造成层级膨胀，二则影响浏览体验
- 策略：第一次点击造成的跳转完成之前无视后续点击产生的跳转请求
- 引入：默认支持

### 附加功能：数据预先加载
- 问题：小程序的page1跳转到page2，到page2的onLoad是存在一个300ms ~ 400ms的延时的，在page2的onLoad中开始获取数据会浪费这个延时
- 策略：在 page1 中预先拿取数据，然后在 page2 中直接使用数据；wepy框架对此有良好的实现，参见[WePY 在小程序性能调优上做出的探究](https://segmentfault.com/a/1190000008975448?winzoom=1)
- 引入：可配，参见“安装”小节

### 效果
无限层级路由策略已在 转转二手交易网 小程序中应用了很长一段时间，欢迎体验：  

<img src="images/zzQrCode.jpg" width=400>  

### 安装
以下按wepy项目进行介绍，非wepy项目，参见 [非wepy项目安装方式](../README.md)

0. `npm install --save fancy-mini`
1. 新建空白页面 pages/curtain/curtain
2. app.wpy
```js
import './appPlugin'
```
3. appPlugin.js
```js
import {registerToThis, registerPageHook, pageRestoreHandler, NavRefine} from 'fancy-mini/lib/wepyKit';
import Navigator from 'fancy-mini/lib/navigate/Navigator';
import {customWxPromisify} from 'fancy-mini/lib/wxPromise';

//无限层级路由方案
Navigator.config({
  enableCurtain: true, //是否开启空白中转策略
  curtainPage: '/pages/curtain/curtain',  //空白中转页

  enableTaintedRestore: true, //是否开启实例覆盖自动恢复策略

  /**
   * 自定义页面数据恢复函数，用于
   * 1. wepy实例覆盖问题，存在两级同路由页面时，前者数据会被后者覆盖，返回时需予以恢复
   * 2. 层级过深时，新开页面会替换前一页面，导致前一页面数据丢失，返回时需予以恢复
   */
  pageRestoreHandler: pageRestoreHandler,

  MAX_LEVEL: 10, //最多同时打开的页面层数

  oriNavOverrides: NavRefine, //自定义覆盖部分/全部底层跳转api，如：此处底层使用wepy定义的路由模块，便于使用wepy提供的prefetch等功能
});
registerPageHook('onUnload', Navigator.onPageUnload);

//wx接口Promise化
let {wxPromise, wxResolve} = (function () {
  let overrides = {  //覆盖wx的部分接口
    navigateTo: Navigator.navigateTo,
    redirectTo: Navigator.redirectTo,
    navigateBack: Navigator.navigateBack,
    reLaunch: Navigator.reLaunch,
    switchTab: Navigator.switchTab,
  };

  return {
    wxPromise: customWxPromisify({overrides, dealFail: false}),
    wxResolve: customWxPromisify({overrides, dealFail: true}),
  }
}());
registerToThis("$wxPromise", wxPromise);
registerToThis("$wxResolve", wxResolve);

export { //导出部分api，方便lib文件使用
  wxPromise,
  wxResolve,
}

```

### 使用
- 所有页面不直接调用wx.navigateTo、wx.redirectTo、wx.navigateBack、wx.switchTab、wx.reLaunch，或wepy对应接口，统一改用this.$wxPromise.navigateTo等对应接口
```js
wx.navigateTo({ url: '/pages/index/index'});//错误，无法确保路由过程全部由自定义模块接管
wepy.navigateTo({ url: '/pages/index/index'});//错误，无法确保路由过程全部由自定义模块接管
this.$wxPromise.navigateTo({ url: '/pages/index/index'}); //正确
```
- 所有页面跳转由this.$wxPromise相应接口触发，而不使用&lt;navigator&gt;元素
- 所有页面若有自定义onUnload函数，需在函数中执行父类相应钩子： super.onUnload && super.onUnload()
```js
export default class extends wepy.page {
	//正确，页面中无onUnload函数，会直接执行父类上的统一钩子
}

export default class extends wepy.page {
	onUnload(){//错误，页面中自定义onUnload函数覆盖了统一钩子，影响路由模块监听返回行为
	
	}
}

export default class extends wepy.page {
	onUnload(){//正确，页面自定义onUnload函数中调用了统一钩子
		super.onUnload && super.onUnload()
	}
}
``` 


### 注意事项
- 要求所有页面遵循上一小节中的代码规范
    - 确保路由过程完全由自定义模块接管
    - 少量未遵循会影响返回逻辑正确性，但一般不会造成页面功能问题
    - 可使用eslint插件强制规范： [eslint-plugin-fancy-mini](https://www.npmjs.com/package/eslint-plugin-fancy-mini)


[返回首页](../README.md)

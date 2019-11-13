# Promise化的小程序API

## 背景
目前小程序API均以回调形式提供，当逻辑较为复杂时会造成回调函数层层嵌套，影响代码可读性和思维清晰性，因而将其转为Promise形式使用。

## 示例
```js
  import {wxPromise, wxResolve} from 'fancy-mini/lib/wxPromise';

  function func0() {
    wxPromise.getSystemInfo().then(res=>{
      console.log('sysInfo:',res);
    })
  }

  async function func1(){
    let sysInfoRes = await wxPromise.getSystemInfo(); //调用wx.getImageInfo，并在success回调中resolve, fail回调中reject
    console.log('sysInfo:', sysInfoRes);
  }

  async function func2(){
    let sysInfoRes = await wxResolve.getSystemInfo(); //调用wx.getSystemInfo，并在success、fail回调中resolve，并在res中添加succeeded字段标记成功/失败
    if (!sysInfoRes.succeeded) //处理失败情形
      console.log('get system info failed');
    console.log('sysInfo:', sysInfoRes);
  }
```

## 自定义二次封装示例
```js
  import {customWxPromisify} from 'fancy-mini/lib/wxPromise';
  import Navigator from 'fancy-mini/lib/navigate/Navigator';	
  
  let overrides = { //覆盖部分API，引入自定义逻辑，以优化性能/实现特定需求
    navigateTo: Navigator.navigateTo,
    redirectTo: Navigator.redirectTo,
    navigateBack: Navigator.navigateBack,
  };
  
  let wxPromise = customWxPromisify({
	overrides,
	dealFail: false, //不处理失败情形，成功时resolve，失败时reject
  });
	
  let wxResolve = customWxPromisify({
	overrides,
	dealFail: true, //处理失败情形，成功失败均resolve，并在返回结果中额外标记res.succeeded=true/false
  });
  
  export{
    wxPromise,
    wxResolve
  } 
  
```

[返回首页](../README.md)
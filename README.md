frontKit
========

highly integrated node-webkit based front-end tools.
CCPS 前端呼叫SDK接入手册
===

## SDK内部基本工作流

1. 调用应用方指定接口，获取appId、token和seatAccount（以备API调用）  
2. 成功获取后立即调用CCPS的登录接口，登录座席，同时，定时重调上述接口刷新appId和token  
3. 座席登录成功后，根据座席类型加载相应的初始化脚本，并初始化webSocket  
4. 脚本动态加载完成，前端webSocket收到初始化状态消息（需要预先设定好消息处理函数）  
5. 初始化完毕，根据业务操作调用呼叫API（呼叫、挂机、置忙...）

上面的逻辑介绍了SDK内部完成初始化所经过的步骤，实际使用中不需要过多地了解内部的具体调用。
如有必要，可以查看源文件（[lib/ccps_phone.js](http://ccps.fangdd.net/lib/ccps_phone.js)）。

下面介绍具体的使用步骤。

## SDK引用与初始化步骤

SDK的封装针对前端不同原理的框架作了考量，实际中需要兼容基于jQuery的和angular两大类系的框架，
因此没有封装DOM操作（例如呼叫按钮的状态控制）。
应用方需根据监听函数接收到的状态对象自己实现呼叫按钮及座席状态的界面显示。

在调用各个API之前，需要 **顺序** 完成以下三个步骤：

**<h3 id="step1">第一步：引入sdk入口脚本</h3>**

```html
    <script src="//ccps.fangdd.net/lib/ccps_phone.js"></script>
```

**<h3 id="step2">第二步：配置监听函数（处理座席、呼叫的状态变化）</h3>**

```javascript
	phoneApi.listen(function(obj) {
		if(obj.type == "phone") {
            // 处理呼叫状态变化
        } else if(obj.type == "seat") {
            // 处理座席状态变化
        }
    }
```

type属性对应呼叫、座席状态两种情况，相应地，obj对象的结构有以下两种情况：

**1. 呼叫状态变化**

```javascript
  {
    type: "phone",
    status: "user_answer"|"user_hangup"|"agent_unanswer"|"agent_hangup"|"agent_alerting"|"agent_answer",
    direction: "in"|"out"|null,
    text: "呼叫"|"取消"|"来电"|"挂机"
  }
```
`status`各值对应描述：

+ __agent_alerting__ 座席响铃  
+ __user_answer__ 外呼时，客户接听  
+ __user_hangup__ 呼入或外呼时，客户挂断  
+ __agent_unanswer__ 座席侧未接听  
+ __agent_hangup__ 座席侧取消外呼  


> **Note!**  
> 由于细节状态比较多，回调参数同时包含了简化版的按钮文本指示状态。    
> 这就存在一次呼叫中两次状态变化对应同一按钮文本的情况，所以在回调处理时需要注意。  
> 目前只有`取消`状态会出现这样的情况，因此，像下面这样处理即可：  
> ```javascript
> if(obj.text == "取消" && obj.status == "agent_answer") {  
>      // 开始呼叫，双条件判断保证不会重复执行  
> }```  

`direction`各值对应描述：

+ __in__ 呼入（来电）    
+ __in__ 外呼  
+ __null__ 无法确定方向，一般是挂机操作  

**2. 座席状态变化**

```javascript
  {
    type: "phone",
    status: "online"|"offline"|"pause"|"idle"
  }
```

`status`各值对应描述：

+ __online__  上线（登录后的默认状态，当离线时此操作有效）  
+ __offline__  离线（无法外呼和呼入）  
+ __pause__  置忙（不接收来电）  
+ __idle__ 置闲（取消置忙，即恢复上线状态）  

_**Note!** 离线后只能有上线操作，其它操作无效。_

**<h3 id="step3">第三步：环境配置及获取token和appId</h3>**

```javascript
    // 开发线使用独立的座席配置，其数据是独立的，这里要配置一下，例如根据当前域名判断是否开发线
	phoneApi.setDevMode(devFlag);   // true/false，不调用则默认为生产环境

    // 获取seatAccount, accessToken和appId，
	phoneApi.init(url, args, function(err, data){   // node回调风格
		// 返回座席相关信息，这里可以不做任何处理
	});
```

**示例**

```javascript
phoneApi.init("/ccps/getAccessToken", {seatAccount: "fangdd-sz"}, function(err, data){
    if(err){
        $scope.$emit(err.type, err.msg);  // err.type = [warn|error]
        return;
    }
    console.log("初始化返回：", data);    // eg. {seatAccount: "fangdd-sz", accessToken: "f018feflow83z8", appid: "f35e523f35b7b4"}
});
```

## 呼叫API

sdk所有API放在全局对象phoneApi中，包含以下常用方法：

### phoneApi#setDevMode(flag)  

座席环境配置，生产线请设置`phoneApi.setDevMode(false)`，或者不调用。

### phoneApi#listen(function callback(obj){...})  

配置监听函数，见上述[步骤1](#step1)。

### phoneApi#getUniqueId()  

获取最近一次呼叫的录音唯一标识。

### phoneApi#getRecordUrl(uuid, function callback(url){...})  

获取录音文件地址，参数uuid为录音唯一标识。  
> _**Note!** 该方法只支持外呼的录音获取_

### phoneApi#getPhoneNum()  

获取本次呼叫的客户号码或来电号码。

### phoneApi#callOut(phoneNum, function callback(err, phoneNum){...})  

外呼。根据当前座席的呼叫状态，该函数同时包含了取消外呼、挂断逻辑判断（内部调用`phoneApi.hangUp()`方法）。  
参数`phoneNum`为电话号码，`callback`回调函数使用node回调风格。

### phoneApi#hangUp()  

取消呼叫或挂机

### phoneApi#getTimerInfo()  

获取本次或最近一次通话计时信息，一个对象，包含以下属性：

- timeStr {String} 通话计时显示时间，格式为hh:mm:ss
- millisec {Number} 通话计时毫秒数
- callStart {Number} 外呼开始时间戳
- callAnswer {Number} 通话开始时间戳
- callEnd {Number} 挂断时间戳

> _**Note!** 计时信息对象是调用时生成的，正在呼叫时，某些属性可能不存在，需要加以判断。_

### phoneApi#setBusy()  

置忙

### phoneApi#setIdel()  

置闲

### phoneApi#setOnline()  

上线

### phoneApi#setOffline()  

离线

### phoneApi#logout()  

退出座席。  
> _**Note!** 考虑到天润的计费方式，sdk内部会绑定页面unload事件，页面关闭前自动退出座席，该方法一般不需要调用。_

## FAQ

1. 呼叫、取消呼叫、挂机这些操作是否可以绑定到独立的按钮？

  操作上是可以的，但是要根据当前呼叫状态禁用相应的按钮，避免重复调用。  
  因为重复调用呼叫或取消可能导致调用出错，甚至断开后续状态的推送，目前第三方呼叫平台还无法保证重复操作的连接稳定性。

2. 怎样判断当前是否是正在呼叫状态？

  phoneApi对象有一个calling属性，指示当前是否处于正在呼叫通话的状态，当该值为false时，
  调用`callOut`方法将取消呼叫（至少一方未接通）或挂断（双方已开始通话）。


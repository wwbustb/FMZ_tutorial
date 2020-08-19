[TOC]
学习本教程之前，需要先学习[FMZ发明者量化平台使用入门](https://www.fmz.com/bbs-topic/4145)和[FMZ量化平台策略编写初级教程](https://www.fmz.com/bbs-topic/4158),并且熟练使用编程语言。**初级教程涉及了最常用的函数，但是还有很多函数和功能没有介绍，本教程也不会覆盖，需要浏览平台API文档自行了解。** 学习完本教程之后，你将能写出更自由、更定制化的策略，FMZ平台只是一个工具。

## 访问交易所原始数据

FMZ平台对所有支持的交易所都进行了封装，为了保持统一性，对单个交易所API支持并不完整。如获取K线一般可以传入K线数量或起始时间，而FMZ平台则是固定的，部分平台支持批量下单，FMZ不支持，等等。所以需要一种直接访问交易所数据的方法。**对于公开接口（如行情），可以使用``HttpQuery``,对于加密接口（牵扯到账户信息），需要使用``IO``。** 具体的传入参数，还要参考相应的交易所API文档。上一个教程介绍了``Info``字段返回了原始信息，但仍然无法解决不支持接口的问题。

### GetRawJSON()

返回最后一次REST API请求返回的原始内容(字符串), 可以用来自己解析扩展信息。

```
function main(){
    var account = exchange.GetAccount() //the account doesn't contain all data returned by the request
    var raw = JSON.parse(exchange.GetRawJSON())//raw data returned by GetAccount()
    Log(raw)
}
```

### HttpQuery() 访问公开接口

访问公开接口，Js可以使用``HttpQuery``，Python可以自行使用相关的包，如``urllib``或``requests``。

HttpQuery默认是GET方法，还支持更多功能，具体查看API文档。

```
var exchangeInfo = JSON.parse(HttpQuery('https://api.binance.com/api/v1/exchangeInfo'))
Log(exchangeInfo)
var ticker = JSON.parse(HttpQuery('https://api.binance.com/api/v1/ticker/24hr'))
var kline = JSON.parse(HttpQuery("https://www.quantinfo.com/API/m/chart/history?symbol=BTC_USD_BITFINEX&resolution=60&from=1525622626&to=1561607596"))
```
Python使用requests例子

```Python
import requests
resp = requests.get('https://www.quantinfo.com/API/m/chart/history?symbol=BTC_USD_BITFINEX&resolution=60&from=1525622626&to=1561607596')
data = resp.json()
```

### IO函数访问加密接口

对于需要API-KEY签名的接口，可以使用IO函数，用户只需要关心传入参数，具体签名过程将由底层完成。

FMZ平台目前不支持BitMEX止损单，按照以下步骤通过IO实现。

- 先找到BitMEX的API接口说明页面：``https://www.bitmex.com/api/explorer/``。
- 找到BitMEX的下单地址为：``https://www.bitmex.com/api/v1/order`` ,方法为``POST``。因为FMZ已经在内部指定了根地址，只需要传入"/api/v1/order"就行了。
- 相应的参数 ``symbol=XBTUSD&side=Buy&orderQty=1&stopPx=4000&ordType=Stop``

具体的代码：

```
var id = exchange.IO("api", "POST", "/api/v1/order", "symbol=XBTUSD&side=Buy&orderQty=1&stopPx=4000&ordType=Stop")
//也可以直接传入对象
var id = exchange.IO("api", "POST", "/api/v1/order", "", JSON.stringify({symbol:"XBTUSD",side:"Buy",orderQty:1,stopPx:4000,ordType:"Stop"}))
```

更多IO的例子：https://www.fmz.com/bbs-topic/3683

## 使用websocket

基本上所有数字货币交易所都支持websocket发送行情，部分交易所支持websocket更新账户信息。相比于rest API, websocket一般具有延时低，频率高，不受平台rest API频率限制等有有点，缺点是有中断问题，处理不直观。

本文将主要介绍在FMZ发明者量化平台,使用JavaScript语言，使用平台封装的Dial函数进行连接，具体说明和参数在文档，搜索Dial，为了实现各种功能，Dial函数进行了几次更新，本文将涵盖这一点，并介绍基于wss的事件驱动的策略，以及连接多交易所问题。Python也可以使用Dial函数，也可以使用相应的库。

### 1.websocket连接
一般直接连接即可，如获取币安全ticker推送：
```
var client = Dial("wss://stream.binance.com:9443/ws/!ticker@arr")
```
对于返回数据是压缩格式，需要在连接是指定，compress指定压缩格式，mode代表发送返回数据那个需要压缩，如连接OKEX：
```
var client = Dial("wss://real.okex.com:10441/websocket?compress=true|compress=gzip_raw&mode=recv")
```
Dial函数支持重连，由底层Go语言完成，检测的连接断开会重连，对于请求数据内容已经在url中的，如刚才币安的例子，很方便，推荐使用。对于需要发送订消息的，可以自己维护重连机制。
```
var client = Dial("wss://stream.binance.com:9443/ws/!ticker@arr|reconnect=true")
```    
订阅wss消息，一些交易所的请求在url中，也有一些需要自己发送订阅的频道，如coinbase:
```
client = Dial("wss://ws-feed.pro.coinbase.com", 60)
client.write('{"type": "subscribe","product_ids": ["BTC-USD"],"channels": ["ticker","heartbeat"]}')
```
### 2.加密接口连接
一般使用websocket读取行情，但也可以用于获取订单、账户推送，这类加密数据的推送有时会有很大延时，需谨慎使用。由于加密方法较为复杂，这里给出几个例子参考。注意到都只需要AccessKey，可以设置成策略参数，如需要SecretKey，可用exchange.HMAC()函数隐式调用，保证安全。


```
//火币期货推送例子
    var ACCESSKEYID = '你的火币账户的accesskey'
    var apiClient = Dial('wss://api.hbdm.com/notification|compress=gzip&mode=recv')
    var date = new Date(); 
    var now_utc =  Date.UTC(date.getUTCFullYear(), date.getUTCMonth(), date.getUTCDate(),date.getUTCHours(), date.getUTCMinutes(), date.getUTCSeconds());
    var utc_date = new Date(now_utc)
    var Timestamp = utc_date.toISOString().substring(0,19)
    var quest = 'GET\napi.hbdm.com\n/notification\n'+'AccessKeyId='+ACCESSKEYID+'&SignatureMethod=HmacSHA256&SignatureVersion=2&Timestamp=' + encodeURIComponent(Timestamp)
    var signature = exchange.HMAC("sha256", "base64", quest, "{{secretkey}}")
    auth = {op: "auth",type: "api",AccessKeyId: ACCESSKEYID, SignatureMethod: "HmacSHA256",SignatureVersion: "2", Timestamp: Timestamp, Signature:encodeURI(signature)}
    apiClient.write(JSON.stringify(auth))
    apiClient.write('{"op": "sub","cid": "orders","topic": "orders.btc'}')
    while (true){
        var data = datastream.read()
        if('op' in data && data.op == 'ping'){
            apiClient.write(JSON.stringify({op:'pong', ts:data.ts}))
        }
    }
    
//币安推送例子，注意需要定时更新listenKey
    var APIKEY = '你的币安accesskey'
    var req = HttpQuery('https://api.binance.com/api/v1/userDataStream','',null,'X-MBX-APIKEY:'+APIKEY);
    var listenKey = JSON.parse(req).listenKey;
    HttpQuery('https://api.binance.com/api/v1/userDataStream', {method:'DELETE',data:'listenKey='+listenKey}, null,'X-MBX-APIKEY:'+APIKEY);
    listenKey = JSON.parse(HttpQuery('https://api.binance.com/api/v1/userDataStream','',null,'X-MBX-APIKEY:'+APIKEY)).listenKey;
    var datastream = Dial("wss://stream.binance.com:9443/ws/"+listenKey+'|reconnect=true',60);
    var update_listenKey_time =  Date.now()/1000;
    while (true){
        if (Date.now()/1000 - update_listenKey_time > 1800){
            update_listenKey_time = Date.now()/1000;
            HttpQuery('https://api.binance.com/api/v1/userDataStream', {method:'PUT',data:'listenKey='+listenKey}, null,'X-MBX-APIKEY:'+APIKEY);
        }
        var data = datastream.read()
    }

//BitMEX推送例子
    var APIKEY = "你的Bitmex API ID"
    var expires = parseInt(Date.now() / 1000) + 10
    var signature = exchange.HMAC("sha256", "hex", "GET/realtime" + expires, "{{secretkey}}")//secretkey在执行时自动替换，不用填写
    var client = Dial("wss://www.bitmex.com/realtime", 60)
    var auth = JSON.stringify({args: [APIKEY, expires, signature], op: "authKeyExpires"})
    var pos = 0
    client.write(auth)
    client.write('{"op": "subscribe", "args": "position"}')
    while (true) {
        bitmexData = client.read()
        if(bitmexData.table == 'position' && pos != parseInt(bitmexData.data[0].currentQty)){
            Log('position change', pos, parseInt(bitmexData.data[0].currentQty), '@')
            pos = parseInt(bitmexData.data[0].currentQty)
        }
    }
```
### 3.websocket读取
一般在死循环中不断读取即可，代码如下：
```
function main() {
    var client = Dial("wss://stream.binance.com:9443/ws/!ticker@arr");
    while (true) {
        var msg = client.read()
        var data = JSON.parse(msg) //把json字符串解析为可引用的object
// 处理data数据
    }
}
``` 
wss数据推送速度很快，Go的底层会把所有的数据缓存在队列中，等程序调用read时，再依次返回。而机器人的下单等操作会带来延时，可能会造成数据的累积。对于成交推送，账户推送，深度插值推送等这类信息，我们需要历史数据，对于行情数据，我们大部分情况只关心最新的，不关心历史数据。

``read()``如果不加参数，会返回最旧的数据，没数据时阻塞到返回。如果想要最新数据，可以用``client.read(-2)``立即返回最新数据,但再没数据时回返回null，需要判断再引用。

根据如何对待缓存的旧数据，以及无数据时是否堵塞，read有不同的参数，具体如下图，看起来很复杂，但让程序更加灵活。
 /upload/asset/2487a91d0da4e4bc7c4.jpg 

### 4.连接多个交易所websocket
对于这种情况程序中显然不能用简单的read(),因为一个交易所会堵塞等待消息，此时另一个交易所即使有新消息也将接收不到。一般处理方式为：
```
    function main() {
        var binance = Dial("wss://stream.binance.com:9443/ws/!ticker@arr");
        var coinbase = Dial("wss://ws-feed.pro.coinbase.com", 60)
        coinbase.write('{"type": "subscribe","product_ids": ["BTC-USD"],"channels": ["ticker","heartbeat"]}')
        while (true) {
            var msgBinance = binance.read(-1) // 参数-1代表无数据立即返回null,不会阻塞到有数据返回
            var msgCoinbase = coinbase.read(-1)
            if(msgBinance){
                // 此时币安有数据返回
            }
            if(msgCoinbase){
                // 此时coinbase有数据返回
            }
            Sleep(1) // 可以休眠1ms
        }
    }
```
### 5.断线重连问题

这部分处理较为麻烦，因为推送数据可能中断，或者推送延时极高，即使能接收到heartbeat也不代表数据还在推送，可以设置一个事件间隔，如果超过间隔没有收到更新就重新连接，并且最好隔一段时间和rest返回的结果对比，看数据是否准确。对于币安这种特殊情况，直接设置自动重连即可。

### 6.使用websocket的一般程序框架
由于已经使用了推送数据，程序自然也要写成事件驱动，注意推送数据频繁，不用过多请求导致被封，一般可以写成：
```
    var tradeTime = Date.now()
    var accountTime = Date.now()
    function trade(data){
        if(Date.now() - tradeTime > 2000){//这里即限制了2s内只交易一次
            tradeTime = Date.now()
            //交易逻辑
        }
    }
    function GetAccount(){
        if(Date.now() - accountTime > 5000){//这里即限制了5s内只获取账户一次
            accountTime = Date.now()
            return exchange.GetAccount()
        }
    }
    function main() {
        var client = Dial("wss://stream.binance.com:9443/ws/!ticker@arr|reconnect=true");
        while (true) {
            var msg = client.read()
            var data = JSON.parse(msg)
            var account = GetAccount()
            trade(data)
        }
    }
```


### 7.总结
各个交易所的websocket的连接方式，数据发送方式，可订阅的内容，数据格式往往不相同，所以平台并没有进行封装，需要用Dial函数自行连接。本文基本涵盖了一些基本的注意事项，如果还有问题，欢迎提问。

PS.一些交易所虽然没有提供websocket行情，但实际上登陆网站使用调式功能，会发现都是使用的websocket推送，研究一下就会发现订阅格式和返回格式。有些看起来像是加密过，用base64解码再解压就可以看到了。

## 多线程并发

JavaScript可以通过Go函数实现并发，Python可以使用相应的多线程库。

在实现量化策略时，很多情况下，并发执行可以降低延时提升效率。以对冲机器人为例，需要获取两个币的深度，顺序执行的代码如下：
```
var depthA = exchanges[0].GetDepth()
var depthB = exchanges[1].GetDepth()
```
请求一次rest API存在延时，假设是100ms，那么两次获取深度的时间实际上不一样，如果需要更多的访问，延时问题将会更突出，影响策略的执行。

JavaScript由于没有多线程，因此底层封装了Go函数解决这个问题，Go函数可用于需要网络访问的API，如``GetDepth``,``GetAccount``等等。也支持``IO``,调用如:``exchange.Go("IO", "api", "POST", "/api/v1/contract_batchorder", "orders_data=" + JSON.stringify(orders))``但由于设计机制，实现起来较为繁琐。
```
var a = exchanges[0].Go("GetDepth")
var b = exchanges[1].Go("GetDepth")
var depthA = a.wait() //调用wait方法等待返回异步获取depth结果 
var depthB = b.wait()
```
在大多数简单情况下，这样写策略并无问题。但注意到每次策略循环都要重复这个过程，中间变量a,b实际上只是临时辅助。如果我们的并发任务非常多，就要另外纪录a和depthA,b和depthB之间的对应关系，当我们的并发任务不确定时，情况就更加复杂。因此，我们希望实现一个函数：当写Go并发时，同时绑定一个变量，当并发运行结果返回时，结果自动赋值给变量，这样就省去了中间变量，使程序更加简洁。具体实现如下：
```
function G(t, ctx, f) {
    return {run:function(){
        f(t.wait(1000), ctx)
    }}
}
```
我们定义了一个G函数，其中参数t是将要执行的Go函数，ctx是记录程序上下文，f为具体赋值的函数。等会就会看到这个函数的作用。

这时，整体的程序框架可以写为类似于“生产者-消费者”模型（有一些区别），生产者不断发出任务，消费者将它们并发执行，一下代码仅为演示，不涉及到程序的执行逻辑。
```
var Info = [{depth:null, account:null}, {depth:null, account:null}] //加入我们需要获取两个交易所的深度和账户，跟多的信息也可以放入，如订单Id，状态等。
var tasks = [ ] //全局的任务列表

function produce(){ //下发各种并发任务
  //这里省略了任务产生的逻辑，仅为演示
  tasks.push({exchange:0, ret:'depth', param:['GetDepth']})
  tasks.push({exchange:1, ret:'depth', param:['GetDepth']})
  tasks.push({exchange:0, ret:'sellID', param:['Buy', Info[0].depth.Asks[0].Price, 10]})
  tasks.push({exchange:1, ret:'buyID', param:['Sell', Info[1].depth.Bids[0].Price, 10]})
}
function worker(){
    var jobs = []
    for(var i=0;i<tasks.length;i++){
        var task = tasks[i]
        jobs.push(G(exchanges[task.exchange].Go.apply(this, task.param), task, function(v, task) {
                    Info[task.exchange][task.ret] = v //这里的v就是并发Go函数wait()的返回值，可以仔细体会下
                }))
    }
    _.each(jobs, function(t){
            t.run() //在这里并发执行所有任务
        })
    tasks = []
}
function main() {
    while(true){
        produce()         // 发出交易指令
        worker()        // 并发执行
        Sleep(1000)
    }
}
```
看上去兜了一圈只实现了一个简单功能，实际上大大简化了代码复杂程度，我们只需关心程序需要产生什么任务，由worker()程序自动将他们并发执行，并返回相应的结果。灵活性提升了很多。

## Chart函数画图

初级教程介绍画图是推荐画图类库，大部分情况下可以满足需求。如果需要更进一步定制，可以直接操作Chart对象。

``Chart({…})``内部参数为HighStock和HighCharts对象，只是额外添加了一个参数`` __isStock``来区分是否是HighStock。HighStock更关注时间序列的图，因此更常用。FMZ基本支持HighCharts和HighStock的基本模块，但不支持额外的modules。

具体的HighCharts例子：https://www.highcharts.com/demo ；HighStock例子： https://www.highcharts.com/stock/demo 。参考这些例子的代码，可以方便移植到FMZ上。

可以调用add([series索引(如0), 数据])向指定索引的series添加数据, 调用reset()清空图表数据, reset可以带一个数字参数, 指定保留的条数。支持显示多个图表, 配置时只需传入数组参数即可如: var chart = Chart([{…}, {…}, {…}]), 比如图表一有两个series, 图表二有一个series, 图表三有一个series, 那么add时指定0与1序列ID代表更新图表1的两个序列的数据, add时指定序列ID为2指图表2的第一个series的数据, 指定序列3指的是图表3的第一个series的数据。

一个具体的例子：
```
var chart = { // 这个 chart 在JS 语言中 是对象， 在使用Chart 函数之前我们需要声明一个配置图表的对象变量chart。
    __isStock: true,                                    // 标记是否为一般图表，有兴趣的可以改成 false 运行看看。
    tooltip: {xDateFormat: '%Y-%m-%d %H:%M:%S, %A'},    // 缩放工具
    title : { text : '差价分析图'},                       // 标题
    rangeSelector: {                                    // 选择范围
        buttons:  [{type: 'hour',count: 1, text: '1h'}, {type: 'hour',count: 3, text: '3h'}, {type: 'hour', count: 8, text: '8h'}, {type: 'all',text: 'All'}],
        selected: 0,
        inputEnabled: false
    },
    xAxis: { type: 'datetime'},                         // 坐标轴横轴 即：x轴， 当前设置的类型是 ：时间
    yAxis : {                                           // 坐标轴纵轴 即：y轴， 默认数值随数据大小调整。
        title: {text: '差价'},                           // 标题
        opposite: false,                                // 是否启用右边纵轴
    },
    series : [                                          // 数据系列，该属性保存的是 各个 数据系列（线， K线图， 标签等..）
        {name : "line1", id : "线1,buy1Price", data : []},  // 索引为0， data 数组内存放的是该索引系列的 数据
        {name : "line2", id : "线2,lastPrice", dashStyle : 'shortdash', data : []}, // 索引为1，设置了dashStyle : 'shortdash' 即：设置 虚线。
    ]
};
function main(){
    var ObjChart = Chart(chart);  // 调用 Chart 函数，初始化 图表。
    ObjChart.reset();             // 清空
    while(true){
        var nowTime = new Date().getTime();   // 获取本次轮询的 时间戳，  即一个 毫秒 的时间戳。用来确定写入到图表的X轴的位置。
        var ticker = _C(exchange.GetTicker);  // 获取行情数据
        var buy1Price = ticker.Buy;           // 从行情数据的返回值取得 买一价
        var lastPrice = ticker.Last + 1;      // 取得最后成交价，为了2条线不重合在一起 ，我们加1
        ObjChart.add([0, [nowTime, buy1Price]]); // 用时间戳作为X值， 买一价 作为Y值 传入 索引0 的数据序列。
        ObjChart.add([1, [nowTime, lastPrice]]); // 同上。
        Sleep(2000);
    }
}
```
一个使用了图表布局的例子：https://www.fmz.com/strategy/136056

## 回测进阶

### Python本地回测

具体开源地址：https://github.com/fmzquant/backtest_python

**安装**

在命令行输入以下命令：
```
pip install https://github.com/fmzquant/backtest_python/archive/master.zip
```
**简单例子**

回测参数在策略代码开头以注释的形式设置，具体见FMZ网站策略编辑界面保存回测设置。
```
'''backtest
start: 2018-02-19 00:00:00
end: 2018-03-22 12:00:00
period: 15m
exchanges: [{"eid":"OKEX","currency":"LTC_BTC","balance":3,"stocks":0}]
'''
from fmz import *
task = VCtx(__doc__) # initialize backtest engine from __doc__
print exchange.GetAccount()
print exchange.GetTicker()
print task.Join() # print backtest result
```
**回测**

由于完整的策略需要死循环，在回测结束后将抛出EOF异常以终止程序，因此需要做好容错。

```
# !/usr/local/bin/python
# -*- coding: UTF-8 -*-

'''backtest
start: 2018-02-19 00:00:00
end: 2018-03-22 12:00:00
period: 15m
exchanges: [{"eid":"Bitfinex","currency":"BTC_USD","balance":10000,"stocks":3}]
'''

from fmz import *
import math
import talib

task = VCtx(__doc__) # initialize backtest engine from __doc__

# ------------------------------ 策略部分开始 --------------------------

print exchange.GetAccount()     # 调用一些接口，打印其返回值。
print exchange.GetTicker()

def adjustFloat(v):             # 策略中自定义的函数
    v = math.floor(v * 1000)
    return v / 1000

def onTick():
    Log("onTick")
    # 具体的策略代码


def main():
    InitAccount = GetAccount()
    while True:
        onTick()
        Sleep(1000)

# ------------------------------ 策略部分结束 --------------------------

try:
    main()                     # 回测结束时会 raise EOFError() 抛出异常，来停止回测的循环。所以要对这个异常处理，在检测到抛出的异常后调用 task.Join() 打印回测结果。
except:
    print task.Join()         
```

### 自定义回测数据

exchange.SetData(arr) , 切换回测数据源，使用自定义的K线数据。参数 arr ，是一个元素为K线柱数据的数组（即：K线数据数组，暂时仅支持 JavaScript 回测。

arr数组中,单个元素的数据格式：
```
[
    1530460800,    // time     时间戳
    2841.5795,     // open     开盘价
    2845.6801,     // high     最高价
    2756.815,      // low      最低价
    2775.557,      // close    收盘价
    137035034      // volume   成交量
]
```
数据源可以放在 “模板类库” 中导入。
```
function init() {                                                          // 模板中的 init 初始化函数会在加载模板时，首先执行，确保 exchange.SetData(arr) 函数先执行，初始化，设置数据给回测系统。
    var arr = [                                                            // 回测的时候需要使用的K线数据
        [1530460800,2841.5795,2845.6801,2756.815,2775.557,137035034],      // 时间最早的一根 K线柱 数据
        ... ,                                                              // K线数据太长，用 ... 表示，数据此处省略。
        [1542556800,2681.8988,2703.5116,2674.1781,2703.5116,231662827]     // 时间最近的一根 K线柱 数据
    ]
    exchange.SetData(arr)                                                  // 导入上述 自定义的数据
    Log("导入数据成功")
}
```
注意：一定要在初始化时，首先导入自定义数据（即调用 exchange.SetData 函数设置数据）, 自定义的K线数据周期必须和回测页面设置的底层K线周期一致，即：自定义的K线数据，一根K线时间是1分钟，那么回测中设置的底层K线周期也要设置为1分钟。

## 使用FMZ不支持的交易所

如果不支持的交易所和已支持的交易所API完全相同，只是基地址不同，可以通过切换基地址的方式支持。具体为添加交易所时选择已支持的交易所，但API-KEY填写不支持的交易所，在策略中用IO切换基地址，如：

```
exchange.IO("base", "http://api.huobi.pro") 
//http://api.huobi.pro为为支持交易所API基地址，注意不用添加/api/v3之类的，会自动补全

```

并不是所有的交易所FMZ都支持，但本平台提供了通用协议的接入方式。具体原理为：

- 自己写代码接入交易所，程序将创建一个网络服务。
- 在FMZ平台添加交易所，指定网络服务的地址和端口。
- 当托管者运行通用协议的交易所的机器人时，策略中的API访问会发送给通用协议。
- 通用协议根据请求访问交易所并返回结果给托管者。
  
简单来说，通用协议相当于一个中介，按照相应的标准代理了托管者的请求并返回数据。通用协议的代码需要自己完成，写出通用协议实际上代表了你可以单独接入交易所，完成策略。FMZ官方有时会发布交易所的通用协议exe版本。通用协议也可以使用Python来完成，这时可以当成一个普通的机器人在托管者上运行。

具体协议的介绍：https://www.fmz.com/bbs-topic/1052
Python写通用协议的例子：https://www.fmz.com/strategy/101399

## 创建自己的量化平台

和交易所各种操作都可以通过API实现一样，FMZ网站也是基于API的，你可以申请自己的FMZ网站API-KEY实现如创建、重启、删除机器人、获取机器人列表、获取机器人日志等各种功能，具体参考API文档“FMZ平台扩展API”部分。

由于FMZ平台的强大扩展性，你可以根据扩展API创建自己的量化平台，让用户在你的平台运行机器人等。具体参考https://www.fmz.com/bbs-topic/1697


## 成为FMZ的合作伙伴

### 推广网易云课堂

数字货币交易市场由于其特殊性越来越受到量化交易者的关注，实际上程序化交易已经是数字货币的主流，对冲做市等策略无时无刻不在活跃着市场。而编程基础薄弱的初学者想要进入这一领域，面对众多的交易所和多变的API，困难重重。发明者（FMZ）量化平台（原BotVs，www.fmz.com)是目前最大的数字货币量化社区和平台，4年多来帮助成千上万的初学者走向了量化交易之路。此课程价格只有20元，面向初学者。

推广[网易云课堂数字货币量化交易课程](https://study.163.com/course/courseMain.htm?share=2&shareId=400000000602076&courseId=1006074239&_trace_c_p_k2_=afd7f12df6be404e93b4e1446e237355)。登陆网易云课堂，分享你的课程链接（链接带独有的courseId），其他人通过此链接注册并购买课程，你将得到50%共10元的分成。关注“网易云课堂精品课推广”微信公众号即可提现。欢迎大家邀请他人，在微博QQ群推广。

### 推广返佣活动

消费者点击推广链接，并且在半年之内注册充值，我司按照有效订单中的有效金额进行返佣。佣金将以积分的形式返还到推广者的账户中，用户可以以10：1的比例兑换发明者量化交易平台账户余额，也可以在以后用积分兑换发明者量化周边商品。活动具体链接：https://www.fmz.com/bbs-topic/3828

### FMZ量化平台企业版

可将完整的FMZ网站部署到企业或团队的专属服务器上，实现完全的控制和功能定制。FMZ网站经过约10万用户的使用和检验，达到了很高的可用性与安全性，可节约量化团队和企业的时间成本。企业版那面向中型量化交易团队、商品期货服务商等，具体报价联系管理员。

### 做市系统

为交易所提供市场流动性与资金管理的专业系统，可能是市场上最完善的做市系统，被很多交易所和团队的使用。

### 交易所方案

发明者科技交易系统采用内存撮合技术，订单处理速度高达 200 万笔 / 秒，能够保证订单处理不会出现任何延迟和卡顿。可保持同时在线用户数量超 2000 万的交易所流畅稳定运行。多层、多集群的系统架构保证了系统的安全性、稳定性、易扩展性。功能部署、版本更新无需停机进行，最大限度保障终端用户的操作体验。目前可以在wex.app模拟交易所体验到这个系统。
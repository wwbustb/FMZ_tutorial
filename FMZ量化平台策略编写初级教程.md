[TOC]
本教程包含策略编写的初级知识，包括API介绍、回测、图表等内容。学习完此基础教程后，用户将能够熟练使用基础的API，编写出稳定的实盘策略。在学习本教程之前，需要先学习[FMZ发明者量化平台使用入门](https://www.fmz.com/bbs-topic/4145)　。

旧版教程：[发明者量化(FMZ.COM)策略编写完全使用手册2.0（教程）](https://www.fmz.com/bbs-topic/458), 这个教程列出了很多帖子索引，也推荐浏览看看。

## 策略编写的初步说明

### API的介绍

程序化交易就是用程序通过API和交易所连接，实现按照设计的意图自动进行买卖或实现其他功能。API全称Application Programming Interface，即应用程序编程接口。

目前数字货币交易所主要有两种接口协议：REST和Websocket。REST协议每获取一次数据，需要访问一次。以模拟交易所wex.app的API为例，直接在浏览器中打开 https://api.wex.app/api/v1/public/ticker?market=BTC_USDT ，得到结果：
```
{"data:{"buy":"11351.73","high":"11595.77","last":"11351.85","low":"11118.45","open":"11358.74","quoteVol":"95995607137.00903936","sell":"11356.02","time":1565593489318,"vol":"3552.5153"}}
```
这就可以看到交易随BTC_USDT交易对的最新行情，每次刷新还会有变化。其中``market=``后面接的是具体交易对参数，可以修改以获取其它交易对数据。对于公开接口，如市场行情，所有人都可以获取到，因此不需要验证，而有些接口如下单和获取账户就需要确定用户身份，这时候就需要使用API-KEY进行签名。Websocket是订阅模式，发送需要订阅的内容之后，交易所会将更新的数据发给程序，不需要每次都重新访问，因此更加高效。

FMZ量化交易平台封装了各个交易所的REST接口，使用统一的方式调用和数据格式，使策略编写更加简单通用。在FMZ平台可以很方便的支持Websocket，将在下篇教程中详细介绍。

### 不同编程语言

FMZ平台API文档大部分以JavaScript为例子，但由于封装，不同语言几乎没有差别，只需要注意语法问题即可。C++稍微特殊，以后的教程会有专门的介绍。由于Js比较简单并且没有兼容性问题，推荐新手使用。FMZ量化平台支持完整的Python，可以自由的安装各种包，推荐有一定编程基础的使用。对于不想学习编程语言，只想快速写出策略的用户，FMZ平台还支持麦语言，基本兼容文华财经策略，有相关经验的推荐使用，缺点是没有编程语言强大灵活。FMZ还支持可视化编程，类似与搭积木的方式实现策略，但不是很推荐，不如代码清晰。由于编程语言相似性很高，不必纠结于选择哪种，都学习一下掌握基础花不了多少精力。

Python由于有不同的版本，可以在程序开头指定，如``#!Python2``,``#!Python3``。注意JavaScript最近升级了ES6语法,有兴趣的可以了解。下面展示了同样功能的Python和Javascript代码，可见仅有语法差异，因此API文档仅给出了Javascript的例子，本教程也会兼顾Python的特殊用例。
````
#python代码
def main():
    while True:
        Log(exchange.GetAccount().Balance)
        Sleep(2000)
#相应的Js代码
function main(){
    while(true){
        Log(exchange.GetAccount().Balance)
        Sleep(2000)
    }
}
````
### 资源推荐

- FMZ平台API文档，本教程不会详细介绍每个接口，具体可查此文档：https://www.fmz.com/api
- Javascript、Python快速入门，编写简单的策略不需要复杂的语法，只需要掌握一些基础概念就行，可以边学编程，边学习本教程： https://www.fmz.com/bbs-topic/382 https://www.fmz.com/bbs-topic/417
- 麦语言文档，对于趋势性策略，麦语言还是很方便的。https://www.fmz.com/bbs-topic/2569
- 旧的FMZ编程指南，内容详细，可作为索引：https://www.fmz.com/bbs-topic/458
- 一个C++调用例子，对C++感兴趣的可以看看，但由于不是解释性语言，调试很困难，不推荐使用：https://www.fmz.com/strategy/61533
- 网易云课堂《数字货币量化交易课程》，FMZ官方出品，仅需20元，内容详细丰富，由浅入深，适合新手入门 [课程链接](https://study.163.com/course/courseMain.htm?share=2&shareId=400000000602076&courseId=1006074239&_trace_c_p_k2_=93d90cb0c1df40da94dbdd8faacd94c5)
- 一些教学策略，适合前期入门，边学习基础边写策略：https://www.fmz.com/square/s:tag:%E6%95%99%E5%AD%A6/1


### 调试工具

FMZ量化平台提供了调试工具供调试API接口，https://www.fmz.com/m/debug 。调试工具只支持JavaScript，只能执行一段时间，不用创建机器人就可以调试实盘接口。return的数据将作为结果返回，调试工具的代码不会保存。在学习本教程中，可以同时使用调试工具进行试验。
 /upload/asset/18c16ddee1c8f7bd96c.jpg  
### 策略程序架构

策略程序和正常的程序一样，按代码顺序执行，特殊之处是必须有一个main函数。由于策略需要不间断运行，通常情况下，需要一个循环加上休眠时间。因为交易所有API访问频率有限制，需要相应的调整休眠时间。这种架构是典型的固定间隔执行，也可以使用websockt来写事件驱动型策略，如只要深度有变化立即执行，将在进阶教程中介绍。

其它有特殊作用的函数如下：
- onexit() 为正常退出扫尾函数，最长执行时间为5分钟，可以不声明，如果超时会报错interrupt错误。可用于退出程序时保存一些结果。
- onerror() 为异常退出函数，最长执行时间为5分钟，可以不声明。
- init() 为初始化函数，策略程序会在开始运行时自动调用，可不声明。

```
function onTick(){
   var ticker = exchange.GetTicker()
   var account = exchange.GetAccount()
    //在这里写策略逻辑，将会每6s调用一次
}
function main(){
    while(true){
        onTick()
        Sleep(6000)
    }
}
```

前面的例子如果网络访问出错可能导致策略直接停止，如果想要一个类似自动重启不会停止的策略，可以再实盘策略用try catch容错主循环（回测不要使用try）。当然只有当策略稳定才建议这样操作，不然会把所有的错误都不报错，难以排查策略的问题。
```
function onTick(){
   var ticker = exchange.GetTicker()
   var account = exchange.GetAccount()
    //在这里写策略逻辑，将会每6s调用一次
}
function main(){
    try{
        while(true){
           onTick()
           Sleep(6000)
       }
    }catch(err){
        Log(err)
    }
}
```

## 交易所API介绍

### 交易所和交易对设置

在调用任何交易所相关的API时，都需要明确交易所和交易对。如果创建机器人时只添加了一个交易所-交易对，``exchange``代表这个对象，如``exchange.GetTicker()``获取的将是这个交易所-交易对的行情ticker。

FMZ平台同时支持添加多个交易所-交易对，如可以同时操作同一个交易所账户的BTC和ETH，也可以同时操作一个交易所的BTC和另一个交易所的ETH。注意同一个交易所不同账户也可以同时添加，它们根据添加到FMZ网站的label区分。当存在多个交易所-交易对时，用``exchanges``数组来表示，按照创建机器人添加顺序分别为``exchanges[0]``、``exchanges[1]``...以此类推。交易对的格式如``BTC_USDT``，前者BTC是交易货币，USDT是计价货币。

 /upload/asset/2737c655475ff43b1ed.jpg  

显然，如果我们操作的交易对很多，这种方式将会很麻烦，此时可以用SetCurrency来切换交易对，如``exchange.SetCurrency("BTC_USDT")``，此时``exchange``所绑定的交易对就变为了``BTC_USDT``，在下次调用改变交易对之前将会一直有效。**注意回测最新支持了切换交易对**。下面是一个具体的例子。

```
var symbols = ["BTC_USDT", "LTC_USDT", "EOS_USDT", "ETH_USDT"]
var buyValue = 1000
function main(){
  for(var i=0;i<symbols.length;i++){
      exchange.SetCurrency(symbols[i])
      var ticker = exchange.GetTicker()
      var amount = _N(buyValue/ticker.Sell, 3)
      exchange.Buy(ticker.Sell, amount)
      Sleep(1000)
  }
}
```

### 获取行情等公开接口

正如前面举得例子，行情接口一般都是公开接口，所有人都能获取。通常的行情接口有：获取行情ticker、获取深度depth、获取K线records、获取成交记录trades。行情是策略进行交易判断的基础，下面将一一介绍，最好能够在调试工具中自己尝试，需要详细的解释可以查看API文档。

各个接口一般都有``Info``字段，表示交易所返回的原始数据字符串，可用于补充额外的信息，用之前需要解析，JavaScript使用``JSON.parse()``,Python使用json库。``Time``字段表示请求的时间戳，可用于判断延时。

**在实盘中使用API接口都有可能访问失败而返回``null``,Python返回``None``，这时在使用其中的数据就会报错并且导致机器人停止，所以容错非常重要。本教程将单独介绍。**

#### GetTicker

获取市场当前行情，大概是最常用的接口，可以查到上一次成交价，买一卖一价，最近成交量等信息。再下单之前可以根据ticker信息来确定交易价格。一个实盘返回的例子``{"Info:{}, "High":5226.69, "Low":5086.37,"Sell":5210.63, "Buy":5208.5, "Last":5208.51, "Volume":1703.1245, "OpenInterest":0, "Time":1554884195976}``。

```
function main() {
    var ticker = exchange.GetTicker()
    Log(ticker) //在调试工具中 return ticker 。可以看到具体的结果。
    Log('上次成交价: ',ticker.Last, '买一价: ', ticker.Buy)
}
```

#### GetDepth

获取挂单深度信息。虽然GetTicker中包含了买一卖一，但如果要查询更深的挂单，可以用这个接口，一般可以查到上下200个挂单。可以使用这个接口计算冲击价格。下面是一个真实的返回结果。其中Asks表示卖单挂单，数组中分别是“卖一”、“卖二”...所以价格也依次上升。Bids表示买单挂单，数组中分别是“买一”、“买二”...价格依次下降。

```
{
    "Info":null,
    "Asks":[
        {"Price":5866.38,"Amount":0.068644},
        {"Price":5866.39,"Amount":0.263985},
        ......
        ]
    "Bids":[
        {"Price":5865.13,"Amount":0.001898},
        {"Price":5865,"Amount":0.085575},
        ......
        ],
    "Time":1530241857399
}
```

使用深度获取买单卖单例子：
```
function main() {
    var depth = exchange.GetDepth()
    Log('买一价个: ', depth.Bids[0].Price, '卖一价格: ', depth.Asks[0].Price)
}
```

#### GetRecords

获取K线，最常用的接口之一，可一次返回较长时间的价格信息，计算各种指标的基础。K线周期如果不指定表示将使用添加机器人时的默认周期。K线长度不能指定，随着时间积累会不断增加，最大2000根，第一次调用大概200根(不同的交易所返回不同）。最后一根K线是最新的K线，所以数据会随着行情不断变化，第一根K线是最旧的数据。

**``exchange.SetMaxBarLen(Len)``可以设置第一次获取K线的数量（部分交易所支持），并且设置了最大的K线数量。** 如：``exchange.SetMaxBarLen(500)``

GetRecords可以指定周期：PERIOD_M1:1分钟,PERIOD_M5:5分钟,PERIOD_M15:15分钟,PERIOD_M30:30分钟,PERIOD_H1:1小时,PERIOD_D1:1天。具体使用为``exchange.GetRecords(PERIOD_M1)``。升级最新的托管者后，会支持自定义周期，直接传周期秒数作为参数就行，分钟级别的自定义会根据1分钟K线合成，1分钟以下K线通过GetTrades()合成，商品期货会根据tick合成，。 **注意教程中还会遇到类似``PERIOD_M1``这种全大写变量，它们是FMZ默认的全局变量，感兴趣的可以自己Log它们的具体的值，平时直接使用就行。**

返回数据实例：
```
[
    {"Time":1526616000000,"Open":7995,"High":8067.65,"Low":7986.6,"Close":8027.22,"Volume":9444676.27669432},
    {"Time":1526619600000,"Open":8019.03,"High":8049.99,"Low":7982.78,"Close":8027,"Volume":5354251.80804935},
    {"Time":1526623200000,"Open":8027.01,"High":8036.41,"Low":7955.24,"Close":7955.39,"Volume":6659842.42025361},
    ......
]
```
迭代K线的例子：
```
function main(){
    var close = []
    var records = exchange.GetRecords(PERIOD_H1)
    Log('total bars: ', records.length)
    for(var i=0;i<records.length;i++){
        close.push(records[i].Close)
    }
    return close
}
```

#### GetTrades

获取一定时间范围的成交数据（不是自己的成交数据），有的交易所不支持。比较不常用，可以在API文档上查询详细介绍。

### 获取账户进行交易

这些接口由于和账户相关，并不能直接获取到，需要使用API-KEY签名。FMZ平台已经后台统一自动处理过，可以直接使用。

#### GetAccount 获取账户

获取账户信息。最常用接口之一，再下单之前需要调用，以避免余额不足。返回结果如：``{"Stocks":0.38594816,"FrozenStocks":0,"Balance":542.858308,"FrozenBalance":0,"Info":{}}``。其中Stocks是交易对的交易货币可用余额，FrozenStocks是可用余额，Balance是计价货币的可用额，FrozenBalance是冻结余额。如交易对是``BTC_USDT``，则Stocks指的是BTC，Balance指的是USDT。

注意到返回结果是指定交易对的结果，交易账户中其它币种的信息在Info字段中，操作多个交易对也不用调用多次。

一个不断打印当前交易对总价值的机器人：

```
function main(){
    while(true){
        var ticker = exchange.GetTicker()
        var account = exchange.GetAccount()
        var price = ticker.Buy
        var stocks = account.Stocks + account.FrozenStocks
        var balance = account.Balance + account.FrozenBalance
        var value = stocks*price + balance
        Log('Account value is: ', value)
        LogProfit(value)
        Sleep(3000)//sleep 3000ms(3s), A loop must has a sleep, or the rate-limit of the exchange will be exceed
        //when run in debug tool, add a break here
    }
}
```

#### Buy 下买单

下买单。调用方式如``exchange.Buy(Price, Amount)``或者``exchange.Buy(Price, Amount, Msg)``,Price是价格，Amount是数量，Msg是一个额外的字符串可以在机器人日志中展示出来，非必须。此方式为挂单，如果无法立即完全成交则会产生未成交订单，下单成功返回结果为订单id，失败为``null``，用于查询订单状态。

如果要下市价买单，Price为-1，Amount为下单价值，如``exchange.Buy(-1, 0.5)``,交易对是``ETH_BTC``，则代表市价买入0.5BTC的ETH。部分交易所不支持市价单，期货回测也不支持。

**部分交易所有价格和数量的精度要求，可用用``_N()``精度函数来控制。对于期货交易Buy和Sell有另外的含义，将单独介绍。**

一个达到相应价格就买入的例子：
```
function main(){
    while(true){
        var ticker = exchange.GetTicker()
        var price = ticker.Sell
        if(price >= 7000){
            exchange.Buy(_N(price+5,2), 1, 'BTC-USDT')
            break
        }
        Sleep(3000)//Sleep 3000ms
    }
    Log('done')
}
```
#### Sell 下卖单

下卖单。参数和Buy相同。市价单的参数意义不同，市价卖单如``exchange.Sell(-1, 0.2)``，代表市价卖出0.2ETH。

#### GetOrder 获取订单

根据订单id获取订单信息。常用接口，调用方式``exchange.GetOrder(OrderId)``,OrderId为订单id，在下单时会返回。**注意订单类型``Type``字段以及订单状态``Status``实际值是数字，代表了不同的意义，但不利于记忆，FMZ用全局的常量代表这些值，如未成交订单的的``Status``的值是0，等同于``ORDER_STATE_PENDING``，所有的这些全局常量可以在文档中查看。**。返回结果：

```
{
    "Id":125723661, //订单id
    "Amount":0.01, //订单数量
    "Price":7000, //订单价格
    "DealAmount":0, //已成交数量
    "AvgPrice":0, //成交均价
    "Status":0, // 0:未完全成交, 1:已成交, 2:已撤单
    "Type":1,// 订单类型，0:买单, 1:卖单
    "ContractType":"",//合约类型，用于期货交易
    "Info":{} //交易所返回原始信息
    }
}
```

一个购买指定数量币种的策略：
```
function main(){
    while(true){
        var amount = exchange.GetAccount().Stocks
        var ticker = exchange.GetTicker()
        var id = null
        if(5-amount>0.01){
            id = exchange.Buy(ticker.Sell, Math.min(5-amount,0.2))
        }else{
            Log('Job completed')
            return //return the main function, bot will stop
        }
        Sleep(3000) //Sleep 3000ms
        if(id){
            var status = exchange.GetOrder(id).Status
            if(status == 0){ //这里也可以用 status == ORDER_STATE_PENDING 来判断。
                exchange.CancelOrder(id)
            }
        }
    }
}
```

#### GetOrders 未成交订单

获取当前交易对所有未成交订单列表。如果没有未完成订单返回空数组。订单列表具体的结果如GetOrder。

撤销当前交易对所有订单的例子：
```
function CancelAll(){
    var orders = exchange.GetOrders()
    for(var i=0;i<orders.length;i++){
        exchange.CancelOrder(orders[i].Id) // cancel order by orderID
    }
}
function main(){
    CancelAll()
    while(true){
        //do something
        Sleep(10000)
    }
}
```

#### CancelOrder 撤单

根据订单id撤销订单。``exchange.CancelOrder(OrderId)``。撤销成功返回true，否则返回false。注意订单已经完全成交会撤销失败。

### 期货和永续合约

数字货币期货交易和现货交易有所不同，上面的现货交易的函数同样适用于期货，单期货交易有专有的函数。在进行数字货币期货程序化交易之前，要在网站上熟悉手工操作，明白基础概念，如开仓、平仓、全仓、逐仓、杠杆、平仓盈亏、浮动收益、保证金等概念以及相应的计算公式，在各个期货交易所都可以找到相关教程，需要自己学习。

永续合约和期货合约类似，不同的是没有同时持有多空的概念。

**如果交易所同时支持期货现货，如OKEX和Huobi期货，需要单独在交易所界面选择“OKEX期货”和“Huobi期货”添加，在FMZ视为和现货不同的交易所。**

#### SetContractType 设置合约

期货交易的第一步就要设置要交易的合约，以OKEX期货为例，创建机器人或回测时选择BTC交易对，还需要在代码中设置是当周、下周还是季度合约。如果不设置会提示``symbol not set``。与现货交易对不同，期货合约往往以BTC或其它交易币种做保证金，交易对添加BTC通常代表以BTC做保证金的BTC_USD交易对，如果存在以USDT做保证金的期货，做需要创建机器人添加BTC_USDT交易对。设置完交易对后，还要设置具体的合约类型，如永续、当周、次周等。

设置完合约后,才能进行进行获取行情,买卖等操作.


```
//OKEX期货
exchange.SetContractType("swap")        // 设置为永续合约
exchange.SetContractType("this_week")   // 设置为当周合约
exchange.SetContractType("next_week")   // 设置为次周合约
exchange.SetContractType("quarter")     // 设置为季度合约

//HuobiDM
exchange.SetContractType("this_week")   // 设置为当周合约 
exchange.SetContractType("next_week")   // 设置为次周合约
exchange.SetContractType("quarter")     // 设置为季度合约

//BitMEX
exchange.SetContractType("XBTUSD")    // 设置为永续合约
exchange.SetContractType("XBTM19")  // 具体某个时间结算的合约，详情登录BitMEX查询各个合约代码

//GateIO
exchange.SetContractType("swap")      // 设置为永续合约，GateIO目前只有永续合约，不设置默认为swap永续合约。 

//Deribit
exchange.SetContractType("BTC-27APR18")  // 具体某个时间结算的合约，详情参看Deribit官网。
```
#### GetPosition 持仓

获取当前持仓信息列表, OKEX(OKCOIN)期货可以传入一个参数, 指定要获取的合约类型。如果没有持仓则返回空列表``[]``。持仓信息返回如下，具体的信息很多，需要结合交易对具体分析。

|数据类型|变量名|说明|
|-|-|-|
|object|Info|交易所返回的原始结构 
|number|MarginLevel|杆杠大小, OKCoin为10或者20,OK期货的全仓模式返回为固定的10, 因为原生API不支持 
|number|Amount|持仓量,OKCoin表示合约的份数(整数且大于1) 
|number|FrozenAmount|仓位冻结量 
|number|Price|持仓均价 
|number|Margin|冻结保证金
|number|Profit|商品期货：持仓盯市盈亏，数字货币：(数字货币单位：BTC/LTC, 传统期货单位:RMB,  注: OKCoin期货全仓情况下指实现盈余, 并非持仓盈亏, 逐仓下指持仓盈亏) 
|const|Type|PD_LONG为多头仓位(CTP中用closebuy_today平仓), PD_SHORT为空头仓位(CTP用closesell_today)平仓, (CTP期货中)PD_LONG_YD为昨日多头仓位(用closebuy平), PD_SHORT_YD为昨日空头仓位(用closesell平) 
|string|ContractType|商品期货为合约代码, 股票为’交易所代码_股票代码’, 具体参数SetContractType的传入类型

```
function main(){
    exchange.SetContractType("this_week");
    var position = exchange.GetPosition();
    if(position.length>0){ //特别要注意引用前要先判断position长度再引用，否则会出错
        Log("Amount:", position[0].Amount, "FrozenAmount:", position[0].FrozenAmount, "Price:",
            position[0].Price, "Profit:", position[0].Profit, "Type:", position[0].Type,"ContractType:", position[0].ContractType)
    }
}
```

#### 期货开仓平仓

首先需要设置杠杆大小，调用方式：``exchange.SetMarginLevel(10)``，10表示10倍杠杆，具体支持的杠杆大小查看相应的交易所，**注意杠杆要在交易所设置，代码里要和交易所设置的一致，否则会出错**。也可以不设置，使用默认杠杆。
然后设置交易方向，调用方式：``exchange.SetDirection(Direction) ``，对应着开仓平仓，**与期货不同，如果永续合约没有同时持有多空概念，即不允许单项持仓，做多时开空会自动平多仓，所有只需要设置``buy``和``sell``即可。如果支持双向持仓，需要设置``closebuy``,``closebuy``。**具体关系：

|操作|SetDirection的参数|下单函数|
|-|-|-|
|开多仓|exchange.SetDirection("buy")|exchange.Buy()|
|平多仓|exchange.SetDirection("closebuy")|exchange.Sell()|
|开空仓|exchange.SetDirection("sell")|exchange.Sell()|
|平空仓|exchange.SetDirection("closesell")|exchange.Buy()|

最后是具体的开仓平仓代码，下单量不同交易所不同，如huobi期货是按张数，一张100美元。注意期货回测不支持市价单。
```
function main(){
    exchange.SetContractType("this_week")    // 举例设置 为OKEX期货 当周合约
    price = exchange.GetTicker().Last
    exchange.SetMarginLevel(10) //设置杠杆为10倍 
    exchange.SetDirection("buy") //设置下单类型为做多 
    exchange.Buy(price+10, 20) // 合约数量为20下单 
    pos = exchange.GetPosition()
    Log(pos)
    Log(exchange.GetOrders()) //查看是否有未成交订单
    exchange.SetDirection("closebuy"); //如果是永续合约，直接设置exchange.SetDirection("sell")
    exchange.Sell(price-10, 20)
}
```

下面给出一个具体的全部平仓的策略例子

```
function main(){
    while(ture){
        var pos = exchange.GetPosition()
        var ticker = exchange.GetTicekr()
        if(!ticker){
            Log('无法获取ticker')
            return
        }
        if(!pos || pos.length == 0 ){
            Log('已无持仓')
            return
        }
        for(var i=0;i<pos.length;i++){
            if(pos[i].Type == PD_LONG){
                exchange.SetContractType(pos[i].ContractType)
                exchange.SetDirection('closebuy')
                exchange.Sell(ticker.Buy, pos[i].Amount - pos[i].FrozenAmount)
            }
            if(pos[i].Type == PD_SHORT){
                exchange.SetContractType(pos[i].ContractType)
                exchange.SetDirection('closesell')
                exchange.Buy(ticker.Sell, pos[i].Amount - pos[i].FrozenAmount)
            }
        }
        var orders = exchange.Getorders()
        Sleep(500)
        for(var j=0;j<orders.length;j++){
            if(orders[i].Status == ORDER_STATE_PENDING){
                exchange.CancelOrder(orders[i].Id)
            }
        }
    }
}

```

### 数字货币杠杆交易

需要在代码里切换为杠杆账户即可，其它与现货交易相同。

使用 exchange.IO("trade_margin") 切换为杠杠账户模式，下单、获取账户资产将访问交易所杠杆接口。
使用 exchange.IO("trade_normal") 切换回普通账户模式。

支持的交易所：

- OKEX V3：杠杆账户模式的交易对和普通的有所不同，有些交易对可能没有。
- 火币：杠杆账户模式的交易对和普通的有所不同，有些交易对可能没有。
- ZB：资金为QC才可转入，杠杆交易板块，不同交易对之间资金独立，即在ETH_QC交易对下的QC币数，在BTC_QC中看不到
- FCoin
- 币安（Binance）

### 商品期货交易

商品期货交易和数字货币期货交易有着很大的不同。首先商品期货的交易时间很短，数字货币24h交易；商品期货的协议也不是常用的REST API；商品期货的交易频率和挂单数量限制，数字货币则很宽松，等等。因此交易商品期货有很多需要特殊注意的地方，建议有丰富的操作手动操作经验。FMZ支持simnow商品期货模拟盘,参考: https://www.fmz.com/bbs-topic/325 。 商品期货公司添加： https://www.fmz.com/bbs-topic/371

商品期货与2019年6月实行了看穿式监管，个人程序化个人用户需要的开户的期货商申请授权码，一般需要4-5天，步骤较为繁琐。FMZ量化平台作为程序化交易提供商向各个期货服务商申请了软件授权码，用户可以无需申请直接使用，在添加期货商是搜索”看穿式“可以看到FMZ已经申请的列表。具体参考帖子： https://www.fmz.com/bbs-topic/3860 。如果你的期货商不再列表中，只能自己申请，或者重新在支持的交易商开户，一般需要2天。FMZ和一些服务商有深入的合作关系，如国泰君安宏源期货，购买了FMZ平台的机构版，可以给用户使用，开户自动成为VIP，并且手续费做到最低。开户参考：https://www.fmz.com/bbs-topic/506 。

由于FMZ平台架构的优势，用户同样可以添加多个期货商账户，并且实现一些其它商品期货程序化交易软件无法完成的功能，如高频tick的合成，参考： https://www.fmz.com/bbs-topic/1184

#### 策略框架


首先由于不是24h交易并且需要登陆操作，在进行交易之前，需要判断链接状态。``exchange.IO("status")``为``true``则表示连接上交易所。如果未登陆成功时调用API，未提示'not login'。可以在策略开始后Sleep(2000)，给登陆一定的时间。

商品期货的行情获取和交易代码与数字货币期货相同，这里将介绍不同和需要注意的地方。

```
function main(){
    _C(exchange.SetContractType,"MA888") //没登陆成功是无法订阅合约的，最好重试一下
    while(true){
        if(exchange.IO("status")){
            var ticker = exchange.GetTicker()
            Log("MA888 ticker:", ticker)
            LogStatus(_D(), "已经连接CTP ！")//_D获取事件
        } else {
            LogStatus(_D(), "未连接CTP ！")
            Sleep(1000)
        }
    }
}
```

建议使用商品期货类库交易（后面有介绍），此时代码会非常简单，不用处理繁琐的细节。源码复制地址：https://www.fmz.com/strategy/57029
```
function main() {
    // 使用了商品期货类库的CTA策略框架
    $.CTA(Symbols, function(st) {
        var r = st.records
        var mp = st.position.amount
        var symbol = st.symbol
        /*
        r为K线, mp为当前品种持仓数量, 正数指多仓, 负数指空仓, 0则不持仓, symbol指品种名称
        返回值如为n: 
            n = 0 : 指全部平仓(不管当前持多持空)
            n > 0 : 如果当前持多仓，则加n个多仓, 如果当前为空仓则平n个空仓,如果n大于当前持仓, 则反手开多仓
            n < 0 : 如果当前持空仓，则加n个空仓, 如果当前为多仓则平n个多仓,如果-n大于当前持仓, 则反手开空仓
            无返回值表示什么也不做
        */
        if (r.length < SlowPeriod) {
            return
        }
        var cross = _Cross(TA.EMA(r, FastPeriod), TA.EMA(r, SlowPeriod));
        if (mp <= 0 && cross > ConfirmPeriod) {
            Log(symbol, "金叉周期", cross, "当前持仓", mp);
            return Lots * (mp < 0 ? 2 : 1)
        } else if (mp >= 0 && cross < -ConfirmPeriod) {
            Log(symbol, "死叉周期", cross, "当前持仓", mp);
            return -Lots * (mp > 0 ? 2 : 1)
        }
    });
}
```

#### CTP数据获取模式

商品期货使用的是CTP协议，所有行情和订单成交都是有变动后才会通知，而查询订单、账户、持仓则是主动查询。所以适合写事件驱动的高频策略。默认模式获取行情的接口如``GetTicker``、``GetDepth``、``GetRecords``都是有缓存的数据才能获取到最新的，没有数据时会一直等待到有数据，所以策略可以不用Sleep。当有行情变化，ticker、depth、records都会被更新，此时调用其中任意接口都会立即返回，被调用过的接口状态被置为等待更新模式，下次调用同样的接口，会等待到有新数据返回。一些冷门合约或者涨跌停情况会出现很长时间不交易情况，这是策略被卡很久也是正常的。

如果想要每次获取行情都能获取到数据，即使是旧数据，可以切换成行情立即更新模式 ``exchange.IO("mode", 0)``。此时策略就不能写为事件驱动了，需要加一个SLeep事件，避免快速的死循环。一些频率不高的策略可以使用这种模式，策略设计简单。使用``exchange.IO("mode", 1)``可以切回默认的缓存模式。

在操作单个合约时，使用默认模式即可。但如果是多个合约，有可能一个合约没有更新行情，导致获取行情接口堵塞，其它合约行情更新也获取不到。要解决这个问题，可以使用立即更新模式，但是不便写高频策略。此时可以使用事件推送模式，获取订单和行情的推送。设置方式为``exchange.IO("wait")``。如果添加了多个交易所对象，这种情况在商品期货中罕见，可以使用``exchange.IO("wait_any")``,此时返回的Index会表明返回的交易所索引。

行情tick变化推送: ``{Event:"tick", Index:交易所索引(按机器人交易所添加顺序), Nano:事件纳秒级时间, Symbol:合约名称}``
订单推送: ``{Event:"order", Index:交易所索引, Nano:事件纳秒级时间, Order:订单信息(与GetOrder获取一致)}``

此时策略结构可以写为：
```
function on_tick(symbol){
    Log("symbol update")
    exchange.SetContractType(symbol)
    Log(exchange.GetTicker())
}

function on_order(order){
    Log("order update", order)
}

function main(){
    while(true){
        if(exchange.IO("status")){ //判断链接状态
            exchange.IO("mode", 0)
            _C(exchange.SetContractType, "MA888")//订阅MA，只有第一次是真正的发出订阅请求，接下来都是程序切换，不耗时间。
            _C(exchange.SetContractType, "rb888")//订阅rb
            while(true){
                var e = exchange.IO("wait")
                if(e){
                    if(e.event == "tick"){
                        on_tick(e.Symbol)
                    }else if(e.event == "order"){
                        on_order(e.Order)
                    }
                }
           }
        }else{
            Sleep(10*1000)
        }
    }
}
```

#### 商品期货与数字货币的差异

另外要注意到商品期货与数字货币交易所的不同。如GetDepth实际上只有一档深度（5档深度收费昂贵），GetTrades也获取不到成交历史（都是根据持仓变化模拟的，没有真实的成交记录）。商品期货有涨跌停限制，涨停时，深度卖单卖一的价格是涨停价格，订单量是0，跌停时，买单买一的价格是跌停价格，订单量是0。另外商品期货查询接口频率，如获取账户、订单、仓位，限制严格，一般要2s一次。商品期货还有下单撤单量的限制等等。


#### 设置合约

exchange.IO("instruments")：返回交易所所有的合约列表{合约名:详情}字典形式,只支持实盘。
exchange.IO("products")：返回交易所所有的产品列表{产品名:详情}字典形式,只支持实盘。
exchange.IO("subscribed")：返回已订阅行情的合约,格式同上,只支持实盘。

传统的CTP期货的``ContractType``就是指的合约ID, 区分大小写。如``exchange.SetContractType("au1506")`` 。合约设置成功后返回合约的详细信息, 如最少一次买多少, 手续费, 交割时间等。订阅多个合约时，只有第一次是真正的发送订阅请求，然后就只是在代码层面切换交易对，不耗时间。主力连续合约为代码为888如MA888, 连续指数合约为000如MA000, 888与000为虚拟合约交易只支持回测, 实盘只支持获取行情。**但是麦语言可以操作主力合约，程序会自动换仓，即平掉非主力仓位，在主力仓位上开新仓。**

未登录成功无法设置合约，但也会立即返回，所以可以用_C重试，知道CTP登陆完成。

#### 开仓平仓

``SetDirection``的Direction可以取``buy, closebuy, sell, closesell``四个参数, 商品期货多出``closebuy_today``与``closesell_today``指平今仓, 默认为``closebuy/closesell``为平昨仓。对于CTP传统期货, 可以设置第二个参数”1”或者”2”或者”3”, 分别指”投机”, “套利”, “套保”, 不设置默认为投机。**具体的买卖、获取仓位、获取订单、撤单、获取账户等操作和数字货币期货交易相同，可参考上一章。**

|操作|SetDirection的参数|下单函数|
|-|-|-|
|开多仓|exchange.SetDirection("buy")|exchange.Buy()|
|平多仓|exchange.SetDirection("closebuy")|exchange.Sell()|
|开空仓|exchange.SetDirection("sell")|exchange.Sell()|
|平空仓|exchange.SetDirection("closesell")|exchange.Buy()|

下面这个例子是具体的平仓函数，注意这个例子过于简单，还要考虑是否处于交易时间、未完全成交如何挂单重试、最大下单量是多少、频率是否过高、具体是滑价还是盘口等等一系列的问题。仅供参考。**实盘的开平仓建议使用平台封装好的类库，https://www.fmz.com/strategy/12961** 。在类库章节有具体介绍，也建议学习一下类库源码。

```
function Cover(contractType, amount, slide) {
    for (var i = 0; i < positions.length; i++) {
        if (positions[i].ContractType != contractType) {
            continue;
        }
        var depth = _C(e.GetDepth);
        if (positions[i].Type == PD_LONG || positions[i].Type == PD_LONG_YD) {
            exchange.SetDirection(positions[i].Type == PD_LONG ? "closebuy_today" : "closebuy");
            exchange.Sell(depth.Bids[0]-slide, amount, contractType, positions[i].Type == PD_LONG ? "平今" : "平昨", 'Bid', depth.Bids[0]);
        } else {
            exchange.SetDirection(positions[i].Type == PD_SHORT ? "closesell_today" : "closesell");
            exchange.Buy(depth.Asks[0]+slide, amount, contractType, positions[i].Type == PD_SHORT ? "平今" : "平昨", 'Ask', depth.Asks[0]);
        }
    }
}
```

商品期货支持自定义订单类型 （支持实盘，回测不支持），以后缀方式指定, 附加在”_“后面比如
```
exchange.SetDirection("buy_ioc");
exchange.SetDirection("sell_gtd-20170111")
```
具体的后缀有：
- ioc	立即完成，否则撤销	THOST_FTDC_TC_IOC
- gfs	本节有效	THOST_FTDC_TC_GFS
- gfd	当日有效	THOST_FTDC_TC_GFD
- gtd	指定日期前有效	THOST_FTDC_TC_GTD
- gtc	撤销前有效	THOST_FTDC_TC_GTC
- gfa	集合竞价有效	THOST_FTDC_TC_GFA

#### 易盛接口

默认在商品期货交易商开通的都是CTP接口，如果有要求的话，可以换为易盛接口。通过FMZ的封装，调用方式相同。差异是账户、订单、持仓都是推送模式，因此托管者本地会维护这些数据，当调用相应接口时会立即返回，不会实际发出请求。

易盛协议自定义订单类型如下：

- gfd	当日有效	TAPI_ORDER_TIMEINFORCE_GFD
- gtc	撤销前有效	TAPI_ORDER_TIMEINFORCE_GTC
- gtd	指定日期前有效	TAPI_ORDER_TIMEINFORCE_GTD
- fak	部分成交，撤销剩余部分	TAPI_ORDER_TIMEINFORCE_FAK
- ioc	立即完成，否则撤销	TAPI_ORDER_TIMEINFORCE_FAK
- fok	未能完全成交，全部撤销	TAPI_ORDER_TIMEINFORCE_FOK

## 常用的全局函数

### Log 日志与微信推送

在机器人界面Log一条日志，字符串后面加上@字符则消息会进入推送队列，绑定微信或telegram后会直接推送。``Log('推送到微信@')``

Log日志的颜色也可以自定义``Log('这是一个红色字体的日志 #ff0000')`` 。``#ff0000``为RGB颜色的16进制表示

所有的日志文件都存在托管者所在目录机器人的sqlit数据库内，可以下载使用数据库软件打开，也可以用于复制备份恢复（数据库名称和机器人id相同）。

### LogProfit 打印收益

记录收益，并且在机器人界面画出收益曲线，机器人重启后也能保留。调用方式：``LogProfit(1000)``。注意``LogProfit``的参数并不一定是收益，可以是任何数字，需要自己填写。

### LogStatus 状态栏展示（含表格）

机器人状态，由于Log日志会保存先来并不断刷新，如果需要一个只展示不保存的信息，可以用``LogStatus``函数。``LogStatus``的参数是字符串，也可以用于表示表格信息。

一个具体的的机器人状态位置展示表格的例子：
```
var table = {type: 'table', title: '持仓信息', cols: ['列1', '列2'], rows: [ ['abc', 'def'], ['ABC', 'support color #ff0000']]}; 
LogStatus('`' + JSON.stringify(table) + '`'); // JSON序列化后两边加上`字符, 视为一个复杂消息格式(当前支持表格) 
LogStatus('第一行消息\n`' + JSON.stringify(table) + '`\n第三行消息'); // 表格信息也可以在多行中出现 
LogStatus('`' + JSON.stringify([table, table]) + '`'); // 支持多个表格同时显示, 将以TAB显示到一组里 
LogStatus('`' + JSON.stringify(tab1) + '`\n' + '`' + JSON.stringify(tab2) + '`\n'); // 上下排列显示多个表
```

### Sleep 休眠

参数为毫秒数,如``Sleep(1000)``为休眠一秒。由于交易所有访问频率限制，一般策略中都要在死循环中加入休眠时间。

### _G 保存数据

机器人重启后，程序会重新开始，如果想保存一些持久信息，``_G``就非常方便实用，可以保存JSON序列化的内容。``_G``函数写在``onexit()``内，这样每次停止策略，会自动保存需要的信息。
如果想要保存更多的格式化数据，_G函数不太适用，可以使用Python直接写入数据库。

```
function onexit(){
    _G('profit', profit)
}
function main(){
    _G("num", 1); // 设置一个全局变量num, 值为1 s
    _G("num", "ok"); // 更改一个全局变量num, 值为字符串ok 
    _G("num", null); // 删除全局变量 num 
    _G("num"); // 返回全局变量num的值,如果不存在返回null

    var profit = 0
    if(_G('profit')){
        profit = _G('profit')
    }
}
```

### _N 精度函数

在下单时，往往要控制价格和数量精度，FMZ内置了_N函数，确定保存小数点位数，如``_N(4.253,2)``结果为4.25。

### _C 自动重试

调用交易所API是，不能保证每次都访问成功，_C是一个自动重试的函数。会一直调用指定函数到成功返回(函数返回null或者false会重试),比如``_C(exchange.GetTicker)``, 默认重试间隔为3秒, 可以调用_CDelay函数来控制重试间隔，比如_CDelay(1000), 指改变_C函数重试间隔为1秒,建议``GetTicker()``,``exchange.GetDepth``,``GetTrade``,``GetRecords``,``GetAccount``,``GetOrders``, ``GetOrder``都是用_C容错，防止访问失败造成程序中断。

``CancelOrder``不能使用_C函数，因为撤单失败有各种原因，如果一个单子已经成交，再撤单会返回失败，使用_C函数会导致一直重试。

_C函数也可以传入参数，也使用于自定义函数。

```
function main(){
    var ticker = _C(exchange.GetTicker)
    var depth = _C(exchange.GetDepth)
    var records = _C(exchange.GetRecords, PERIOD_D1) //传入参数
}

```

### _D 日期函数

直接调用``_D()``返回当前时间字符串，如：``2019-08-15 03:46:14``。如果是回测中调用则返回回测时间。可以使用_D函数判断时间，如：`` _D().slice(11) > '09:00:00':``

``_D(timestamp,fmt)``,则会将ms时间戳转为时间字符串，如``_D(1565855310002)``。fmt参数为时间格式，默认为``yyyy-MM-dd hh:mm:ss``


### TA 指标函数

对于一些常用的指标函数，如MA\MACD\KDJ\BOLL等常用指标，FMZ平台直接内置，具体的支持的指标可查API文档。

使用指标函数之前，最好要判断K线长度。当前面的K线长度无法满足计算所需周期时，结果为``null``。如输入的K线长度为100，计算MA的周期为10，则前9个值都是null，后面才正常计算。

JavaScript也支持完整的talib，作为第三方库支持，调用如``talib.CCI(records)``。参考 http://ta-lib.org/function.html 。对于Python可以自行安装talib库，由于需要编译，不能简单的使用pip安装，可自行搜索安装方式。

**指标函数除了传入K线数据外，还可以传入任意数组**

```
function main(){
    var records = exchange.GetRecords(PERIOD_M30)
    if (records && records.length > 9) {
        var ma = TA.MA(records, 14)
        Log(ma)
    }
}
```

### JavaScript常用函数

这里介绍一些实盘常用的JavaScript函数。

- ``Date.now()`` 返回当前时间戳
- ``parseFloat()`` 将字符串转为数字，如``parseFloat("123.21")``
- ``parseInt()`` 将字符串转为整数
- ``num.toString()`` 将数字转为字符串，num为数字变量
- ``JSON.parse()`` 格式化Json字符串，如``JSON.parse(exchange.GetRawJSON())``
- JavaScript自带Math库函数如``Math.max()``,``Math.abs()``等等常用数学操作，参考： https://www.w3school.com.cn/jsref/jsref_obj_math.asp
- FMZ引用的JavaScript第三方math库，参考：https://mathjs.org/
- FMZ引用的JavaScript第三方underscore库，推荐了解下，方便了很多Js繁琐的操作，参考:https://underscorejs.org/

## 模板类库

写出一个实盘策略功能需要考虑的情况非常多，比如买入5个币这么一个简单的功能，我们要考虑到：当前的余额足够吗？下单的价格是多少？精度是多少？需不需要拆分订单避免冲击市场？未完成订单如何处理？等等细节。在不同的策略中，这些功能是同样的，可以做成一个模板。仿照官方模板，用户也可以自己写模板策略。这里将介绍FMZ官方出的几个十分常用的模板类库，方便用户快速写出自己的策略。

JavaScript数字货币交易类库和商品期货交易类库是默认内置的，不用复制。其它模板类库在策略广场可以找到 https://www.fmz.com/square/20/1 。将模板类库复制并保存，并且在创建自己策略时勾选要使用的类库就可以使用了。

JavaScript模板函数都以``$``开头，Python以``ext``开头。

### 数字货币交易类库

源代码地址：https://www.fmz.com/strategy/10989 ，已经内置，无需复制。具体函数的实现方法可以直接参考源码。

**获取账户：**
```
$.GetAccount(e)

Log($.GetAccount()); // 获取账户信息, 带容错功能
Log($.GetAcccount(exchanges[1]));
```
**下单撤单：**
```
$.Buy/Sell(e, amount)
$.Buy(0.3); // 主交易所买入0.3个币
$.Sell(0.2); // 主交易所卖出0.2个币
$.Sell(exchanges[1], 0.1); // 次交易所卖出0.1个币
$.CancelPendingOrders(e, orderType)

$.CancelPendingOrders(); // 取消主交易所所有委托单
$.CancelPendingOrders(ORDER_TYPE_BUY); // 取消主交易所所有的买单
$.CancelPendingOrders(exchanges[1]); // 取消第二个交易所所有订单
$.CancelPendingOrders(exchanges[1], ORDER_TYPE_SELL); // 取消第二个交易所所有的卖单
```
**判断交叉：**
```
$.Cross(periodA, periodB) / $.Cross(arr1, arr2);

var n = $.Cross(15, 30);
var m = $.Cross([1,2,3,2.8,3.5], [3,1.9,2,5,0.6])
如果 n 等于 0, 指刚好15周期的EMA与30周期的EMA当前价格相等
如果 n 大于 0, 比如 5, 指15周期的EMA上穿了30周期的EMA 5个周期(Bar)
如果 n 小于 0, 比如 -12, 指15周期的EMA下穿了30周期的EMA 12个周期(Bar)
如果传给Cross不是数组, 则函数自动获取K线进行均线计算
如果传给Cross的是数组, 则直接进行比较
```

$.withdraw(e, currency, address, amount, fee, password) 提现函数:
```
$.withdraw(exchange, "btc", "0x.........", 1.0, 0.0001, "***")
```
### 商品期货交易类库

商品期货交易类库使用稳定，推荐使用。源代码地址： https://www.fmz.com/strategy/12961  。已经内置，无需复制。

**CTA库**

- 实盘会自动把指数映射到主力连续
- 会自动处理移仓
- 回测可以指定映射比如 rb000/rb888 就是把rb指数交易映射到主力连续
- 也可以映射到别的合约, 比如rb000/MA888 就是看rb指数的K线来交易MA主力连续

```
function main() {
    $.CTA("rb000,M000", function(r, mp) {
        if (r.length < 20) {
            return
        }
        var emaSlow = TA.EMA(r, 20)
        var emaFast = TA.EMA(r, 5)
        var cross = $.Cross(emaFast, emaSlow);
        if (mp <= 0 && cross > 2) {
            Log("金叉周期", cross, "当前持仓", mp);
            return 1
        } else if (mp >= 0 && cross < -2) {
            Log("死叉周期", cross, "当前持仓", mp);
            return -1
        }
    });
}
```
**类库调用举例**
```
function main() {
    var p = $.NewPositionManager();
    p.OpenShort("MA609", 1);
    p.OpenShort("MA701", 1);
    Log(p.GetPosition("MA609", PD_SHORT));
    Log(p.GetAccount());
    Log(p.Account());
    Sleep(60000 * 10);
    p.CoverAll("MA609");
    LogProfit(p.Profit());
    Log($.IsTrading("MA609"));
    // 多品种时使用交易队列来完成非阻塞的交易任务
    var q = $.NewTaskQueue();
    q.pushTask(exchange, "MA701", "buy", 3, function(task, ret) {
        Log(task.desc, ret)
    })
    while (true) {
        // 在空闲时调用poll来完成未完成的任务
        q.poll()
        Sleep(1000)
    }
}
```

### 画图类库

由于原始的画图函数比较复杂，将在下个教程介绍，新手推荐直接使用画图类库，非常简单的画折线图、K线图等。FMZ内置了简单的类库，可以在策略编辑页面看到，如果没有内置，需要用户自己复制并保存后才能在策略里勾选引用。

 /upload/asset/1884e44f0a24a114b30.png 

Javascript版画图类库复制地址：https://www.fmz.com/strategy/27293
Python版画线类库复制地址：https://www.fmz.com/strategy/39066

具体例子：
```
function main() {
    while (true) {
        var ticker = exchange.GetTicker()
        if (ticker) {
            $.PlotLine('Last', ticker.Last) //可以同时画两条线，Last是这条线的名字
            $.PlotLine('Buy', ticker.Buy)
        }
        Sleep(6000)
    }
}
```

## 策略参数设置

在策略编辑下方有策略参数设置，相当于策略的全局变量，可以在代码的任意位置访问到。策略参数可以在机器人界面修改，重启后生效。因此可将一些变量设置为参数，不用修改策略也能改变参数。
  /upload/asset/13410b347caff4cad1a.jpg 
- **变量名**：即上图中的 number、string、combox等，在策略组可以直接使用。
- **描述** ：参数在策略界面上的名字，便于理解参数的意义。
- **备注** ：参数的详细解释，该描述会在鼠标停留在参数上时 相应的显示出。
- **类型** ：该参数的类型，以下详细介绍。
- **默认值** ：该参数的默认值。

字符串类型和数字类型很容易理解，也是最常用的类型。下拉框将在参数界面展示可选项的下拉框，如可设置下拉框SYMBOL参数值为``BTC|USDT|ETH``，在参数页面下拉中选择了USDT，则策略中SYMBOL的值为USDT的索引1。勾选项是一个可选框，勾上为true，否则为false。

参数还有很多可供设置，参考 https://www.fmz.com/bbs-topic/1306

## 策略回测

当完成一个策略的量化工作后，可以用历史数据来测试您的策略，看看您的策略在历史数据中盈利如何。当然回测结果仅供作为参考。FMZ量化平台支持数字货币现货、期货、BitMEX永续合约、商品期货的回测，其中数字货币主要支持主流品种。
Javascript回测在浏览器进行，Python回测需要在托管者上，可使用平台提供公用托管者。麦语言的回测又更多的参数需要设置，具体参考麦语言文档。

### 回测机制

onbar回测机制是基于K线的，即每一个K线产生一个回测时间点，在此时间点上可以获取到当前K线的高开低收价格、交易量等信息，以及此时间点之前的历史K线信息。这种机制的弊端很明显：在一根K线上，只能产生一次买卖，通常依据的价格是K线的收盘价。并且一根K线只能获取到高开低收四个价格，至于在一根K线内价格如何变化的，是最高价先发生、还是最低价先发生等等信息都无从获取。以1小时K线为例，实盘时肯定每隔几秒获取一次行情信息，交易指令也会在盘中发出而不是等待K线结束。onbar回测机制的好处是易于理解，回测速度极快。

FMZ平台回测分模拟级回测和实盘级回测两种。模拟级回测根据底层K线周期生成模拟的tick，每个底层K线周期上将生成14个回测时间点，**而实盘级则是真实收集的tick，大约几秒就有一次，目前部分支持了真实的深度（包含20档），真实的逐笔成交。** 数据量很大，回测速度慢，因此不能回测特别长的时间。FMZ的回测机制可以使策略在一根K线上交易多次，避免了只能收盘价成交的情况，更加精准又兼顾了回测速度。具体的说明可参考：https://www.fmz.com/digest-topic/4009

回测的策略框架和实盘相同，都是一个死循环。由于回测是在不同回测点上跳跃，此时可以不用Sleep，在一个循环结束会自动跳到下一个时间点。但Python由于程序机制，需要强制一个``Sleep(10)``,以避免卡死。

### 回测的撮合

回测引擎会根据用户下单价和回测时间点的盘口价格撮合，如果买价高于卖一，以卖一成交。如果无法成交，就会产生挂单。要保证成交需要加滑点。如果回测时发生了开不了仓或平不掉的情况，检查是不是有未成交订单造成的仓位冻结。

### 回测页面设置
  /upload/asset/242ed2d259052fd4e4b.jpg  
- 1.回测页面的选择，左侧是策略编辑页面。
- 2.回测起始结束时间，由于数据不完整，回测可能直接从有数据的时间开始。
- 3.回测``GetRecords()``函数的默认周期，也可以在代码中指定周期参数。
- 4.回测机制的选择。
- 5.展示或隐藏跟多回测设置。
- 6.最大日志数、收益数据数、图表数据数等，为了防止数据量过大导致浏览器卡死。
- 6.底层tick生成依据K线周期。
- 7.交易滑点。
- 7.容错，会模拟API请求出错情况，检查策略容错能力。
- 8.是否绘制行情图标，回测中如果使用了TA指标函数，会自展示在图标上，买卖也会标记。
- 9.手续费设置
- 10.添加交易所-交易对和资产。
- 11.回测参数设置，如果参数是数字还支持一键优化参数，自动按照一定范围遍历参数回测。
  
### 回测与实盘的不同

- 1.回测时有效的行情只有GetTicker和GetRecords，其它如获深度、成交历史都不是真实的（因为数据量太大，实盘级回测目前已经支持这些数据，但只有最近数据）。
- 2.回测添加的交易所都是独立账户，目前不支持切换交易对。因此无法在一个账户里操作两个交易对。
- 3.回测中无法使用网络请求。
- 4.回测无法使用IO扩展，只能操作最基础的API。
- 5.回测只能获取标准的数据，像Info之类的牵扯到实盘的数据不存在。
- 6.回测中也有可能不成交，注意冻结订单情况。
- 7.商品期货回测不支持市价单。

## 策略容错与常见错误

前面说过在实盘中使用API接口都有可能访问失败而返回``null``，这时在使用其中的数据就会报错并且导致机器人停止，所以策略要做好容错。

### 常用容错方式

常见错误原因：

- API访问网络错误，接口访问超时将返回null，此时使用就会报错。
- 交易所限制错误，如ip限制、下单精度、访问频率、参数错误、资产不足、市场不能交易、撤销已成交订单等等。可具体根据错误代码查询API文档。
- 交易所返回数据错误，偶有发生，如返回空的深度、延时的账户信息、延时的订单状态等。
- 程序逻辑错误。

在使用API返回数据之前，都要对其是否为null进行判断，下面将介绍集中常用方法：

```
//1.判断为null进行处理
var ticker = exchange.GetTicker();
while(ticker == null){
     Log('ticker 获取出错');
     ticker = exchange.GetTicker();
 }
 Log(ticker.Last);
 // 2.判断不为null再进行引用
 var ticker = exchange.GetTicker();
 if(!ticker){
     Log(ticker.Last);
 }
 // 3._C()函数重试
 var ticker = _C(exchange.GetTicker);
 Log(ticker.Last);
 // 4. try catch容错
 try{
     var ticker = exchange.GetTicker();
     Log(ticker.Last);
 }
 catch(err){
     Log('ticker 获取出错');
 } 

```
如果想要获取错误信息，可以使用``GetLastError()``，将返回上一次出错信息字符串，可以对错误进行差异处理。

### FAQ

论坛置顶帖有许多常见错误汇总： https://www.fmz.com/bbs-topic/1427 。这里将摘要一些，遇到问题可以ctrl+F搜索以下。

> 如何布置托管者？

在添加托管者一节有详细介绍

> 能不能找人代写策略？

https://www.fmz.com/markets 上有一些人提供代写服务，或者在群里咨询，需要自己联系，自担风险。

> 访问所有接口都提示timeout 

是指访问交易所接口超时，如果偶尔出现不是问题，如果一直提示做说明所在网络无法访问，需要使用海外服务器。

> ERR_INVALID_POSITION 错误

回测系统报错，一般为策略编写错误，在没有持仓或者持仓数量不足时，尝试下单平仓，会引起该报错。

> symbol not set

期货交易所回测，代码中没有设置合约， 参看 exchange.SetContractType 函数

> BITMEX 429错误，{"error":{"message":"Rate limit exceeded retry in 1 seconds ……"}}

访问交易所接口频率过高。

> {"status":6004,"msg":"timestamp is out of range"}

服务器时间戳超出范围需要更新服务器时间，不能偏差过大

> GetOrder(455284455): Error: invalid order id or order cancelled. 

些交易所订单取消，交易所就不在维护这个订单信息，无法获取。

> GetOrders: 400: {"code":-1121,"msg":"Invalid symbol."}

无效的交易对，检查下是不是交易对设置错误。

> Secret key decrypt failed

API KEY 解析失败，如果配置了APIKEY后修改过FMZ密码，尝试在FMZ添加交易所页面，重新配置交易所APIKEY。

> Signature not valid: Invalid submission time or incorrect time format [无效的提交时间，或时间格式错误] 

建议使用Linux服务器，或者在这些出现该问题的windows系统安装时间同步软件。

> 为什么设置了全局代理，托管者任然无法访问交易所API？

全局代理并没有代理托管者网络端口，由于延迟问题，最好部署海外服务器的托管者

> 策略如何保存在本地，而不是上传的FMZ上？

使用Python可以导入本地的文件，把正常根据FMZ的API写的策略保存成文件放在自己的服务器上的执行路径下，直接读取执行就可以了。

```
#!python2.7

def run(runfile):
      with open(runfile,"r") as f:
            exec(f.read())
            
def main():
    run('my.py')
```

> 如何使用交易所的测试网或者更改API基地址

使用exchange.SetBase()直接切换到相应的API基地址即可。如：

```
exchange.SetBase("https://www.okex.me")
```


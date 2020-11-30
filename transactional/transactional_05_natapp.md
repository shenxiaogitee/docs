#### 内网穿透

![](https://gitee.com/enioy/img/raw/master/K8S/20201128150314.png) 



![](https://gitee.com/enioy/img/raw/master/K8S/20201128150402.png) 





![](https://gitee.com/enioy/img/raw/master/K8S/20201130132932.png)  





![](https://gitee.com/enioy/img/raw/master/K8S/20201130132049.png)

 

![](https://gitee.com/enioy/img/raw/master/K8S/20201130134257.png) 



收单

• 1、订单在支付页，不支付，一直刷新，订单过期了才支付，订单状态改为已支付了，但是库 存解锁了。

• 使用支付宝自动收单功能解决。只要一段时间不支付，就不能支付了。
 • 2、由于时延等问题。订单解锁完成，正在解锁库存的时候，异步通知才到

• 订单解锁，手动调用收单
 • 3、网络阻塞问题，订单支付成功的异步通知一直不到达

• 查询订单列表时，ajax获取当前未支付的订单状态，查询订单状态时，再获取一下支付宝 此订单的状态

• 4、其他各种问题
 • 每天晚上闲时下载支付宝对账单，一一进行对账




### Redis集群

#### 1.Redis集群方式

##### 1.数据分区方案

###### 1.客户端分区

![](https://gitee.com/enioy/img/raw/master/K8S/20201226110058.png) 

![](https://gitee.com/enioy/img/raw/master/K8S/20201226110150.png) 



###### 2.代理分区

![](https://gitee.com/enioy/img/raw/master/K8S/20201226110236.png)  



###### 3.redis-cluster

详见下章

##### 2.高可用方式

###### 1.Sentinel

![](https://gitee.com/enioy/img/raw/master/K8S/20201226110345.png) 

![](https://gitee.com/enioy/img/raw/master/K8S/20201226110428.png) 

![](https://gitee.com/enioy/img/raw/master/K8S/20201226110532.png) 

![](https://gitee.com/enioy/img/raw/master/K8S/20201226110606.png) 

###### 2.redis-cluster

详见下章



#### 2.Redis-Cluster

![](https://gitee.com/enioy/img/raw/master/K8S/20201226110721.png) 

![](https://gitee.com/enioy/img/raw/master/K8S/20201226110846.png) 

![](https://gitee.com/enioy/img/raw/master/K8S/20201226111130.png) 

![](https://gitee.com/enioy/img/raw/master/K8S/20201226111145.png) 



![](https://gitee.com/enioy/img/raw/master/K8S/20201226111450.png) 

![](https://gitee.com/enioy/img/raw/master/K8S/20201226111608.png) 



#### 3.部署Cluster

![](https://gitee.com/enioy/img/raw/master/K8S/20201226111754.png) 

![](https://gitee.com/enioy/img/raw/master/K8S/20201226112141.png) 

![](https://gitee.com/enioy/img/raw/master/K8S/20201226112255.png) 

![](https://gitee.com/enioy/img/raw/master/K8S/20201226112516.png) 

![](https://gitee.com/enioy/img/raw/master/K8S/20201226112538.png) 

![](https://gitee.com/enioy/img/raw/master/K8S/20201226112733.png) 

![](https://gitee.com/enioy/img/raw/master/K8S/20201226112936.png) 

测试

![](https://gitee.com/enioy/img/raw/master/K8S/20201226113053.png) 

![](https://gitee.com/enioy/img/raw/master/K8S/20201226113225.png) 

 ![](https://gitee.com/enioy/img/raw/master/K8S/20201226113333.png) 

![](https://gitee.com/enioy/img/raw/master/K8S/20201226113429.png) 

![](/Users/SXW/Library/Application Support/typora-user-images/image-20201226113516923.png)

模拟7001宕机 

![](https://gitee.com/enioy/img/raw/master/K8S/20201226113612.png) 

![](https://gitee.com/enioy/img/raw/master/K8S/20201226113947.png) 

7006变成主节点



重新启动7001

![](https://gitee.com/enioy/img/raw/master/K8S/20201226114105.png) 

7001变成从节点



#### 4.k8s部署


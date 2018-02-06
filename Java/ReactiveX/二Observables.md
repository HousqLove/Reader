# 二 Observable
 在Rx中一个观察者（Observer）订阅一个可观察对象（Observable）。观察者对Observable发射的数据做出响应。这种模式极大地简化并发操作，因为创建了一个处于待命状态的观察者，在未来某个时刻响应Observable的通知，而不需要阻塞等待Observable发送数据。  

 经典示意图：  
 ![Image]()
首先介绍BOM：    
#### 什么是BOM #### 
Browser Object Model，即 浏览器对象模型 。
BOM有一个核心对象window，window对象包含了6大核心模块，分别是：    
document对象：即文档对象

frames：即HTML自框架

history：即页面的历史记录

location：即当前页面的地址

navigator：包含浏览器相关信息

screen：用户显示屏幕相关属性
以上出自链接：https://www.jianshu.com/p/0c8b34111e95的部分内容


进入正题：浏览器怎么和服务器通信，就要用到##### http协议 #####，http是基于TCP/IP（传输层）的应用层协议。    

### HTTP协议是无状态的 ###
HTTP协议是无状态的，指的是协议对于事务处理没有记忆能力，服务器不知道客户端是什么状态。也就是说，打开一个服务器上的网页和上一次打开这个服务器上的网页    
之间没有任何联系。HTTP是一个无状态的面向连接的协议，无状态不代表HTTP不能保持TCP连接，更不能代表HTTP使用的是UDP协议（无连接）。

### 长连接和短连接 ###

http1.0是短连接，http1.1是长连接（并非应用层http的长短连接，而是传输层TCP连接的长短连接，因为HTTP是无状态的，浏览器和服务器每进行一次HTTP操作，    
就建立一次连接，但任务结束就中断连接。），所谓长连接和短连接的由来：    
HTTP协议是基于请求/响应模式的（HTTP请求和HTTP响应，都是通过TCP连接这个通道来回传输的），因此只要服务端给了响应，本次HTTP连接就结束了，    
或者更准确的说，是本次HTTP请求就结束了，如果是长连接，TCP连接还没有中断，还可以供其他的http请求和响应来传输数据。    
长连接并不是永久连接的，如果一段时间内（具体的时间长短，是可以在header当中进行设置的，也就是所谓的超时时间），这个连接没有HTTP请求发出的话，    
那么这个长连接就会被断掉。

关于长短连接，这篇文章介绍通俗易懂：https://blog.csdn.net/maquealone/article/details/87345795

##### 长短连接的选择 #####
长连接可以省去较多的TCP建立和关闭的操作，减少浪费，节约时间。对于频繁请求资源的客户来说，较适用长连接。不过这里存在一个问题，存活功能的探测周期太长，还有就是它只是探测TCP连接的存活，属于比较斯文的做法，遇到恶意的连接时，保活功能就不够使了。在长连接的应用场景下，client端一般不会主动关闭它们之间的连接，Client与server之间的连接如果一直不关闭的话，会存在一个问题，随着客户端连接越来越多，server早晚有扛不住的时候，这时候server端需要采取一些策略，如关闭一些长时间没有读写事件发生的连接，这样可 以避免一些恶意连接导致server端服务受损；如果条件再允许就可以以客户端机器为颗粒度，限制每个客户端的最大长连接数，这样可以完全避免某个蛋疼的客户端连累后端服务。
出自：https://www.cnblogs.com/gotodsp/p/6366163.html   

### cookie，session，localStorage，sessionStorage ###
由于http的连接是无状态的特性，所以有了以上四种存储数据的方式。

cookie存储在客户端浏览器，session存储在服务端，session的存在在某种程度上依赖于cookie（个人理解这种说法不完全正确，因为在使用session的客户端存储可以使用cookie，也可以不使用cookie，比如可以使用token字段，使用cookie的时候可以在这说）。

cookie又有httponly类型的cookie，一般由服务端直接设置，并设置httponly属性，这样的cookie在浏览器端使用document.cookie获取不到，避免了篡改cookie，不管是httponly类型的cookie，还是document.cookie设置的cookie，浏览器像服务器发送请求时都会通过Request Header的Cookie字段带给服务端。

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


进入正题：浏览器怎么和服务器通信，就要用到http协议，http是基于TCP/IP（传输层）的应用层协议。   
 
# HTTP协议简介
HTTP（HyperText Transport Protocol）是超文本传输协议的缩写，协议即一种规范，浏览器和服务器之间进行“沟通”的一种规范，HTTP协议采用了请求/响应模型。

HTTP协议包含http协议的请求，和http协议的响应    
http请求包含：请求行（包含请求方法，URL，请求协议/版本），请求头，请求正文（请求体）
http响应包含：状态行，响应头，响应正文

客户端向服务器发送一个请求，请求头包含请求的方法、URL、协议版本、以及包含请求修饰符、客户信息和内容的类似于MIME的消息结构。服务器以一个状态行作为响应，响应的内容包括消息协议的版本，成功或者错误编码加上包含服务器信息、实体元信息以及可能的实体内容。

#### 一次完整的http请求包含以下步骤
（1）对www.baidu.com这个网址进行DNS域名解析，得到对应的IP地址；        
（2）根据这个IP，找到对应的服务器，发起TCP的三次握手；    
（3）建立TCP连接后发起HTTP请求；    
（4）服务器响应HTTP请求，浏览器得到html代码；    
（5）浏览器解析html代码，并请求html代码中的资源（如js、css、图片等）（先得到html代码，才能去找这些资源）；    
（6）浏览器对页面进行渲染呈现给用户；    
（7）服务器关闭关闭TCP连接；     
转自：https://www.cnblogs.com/WindSun/p/11489356.html 里面有详细介绍，关于域名解析等。

#### 请求头和响应头介绍 ####
##### Headers 通用信息头：
（1）Request URL：请求的地址    
（2）Request Method：请求的方法类型    
（3）Status Code：响应状态码    
（4）Remote Address：表示远程服务器地址    
##### 请求(客户端->服务端[request]) ：     
（1）请求行：GET(请求的方式) /newcoder/hello.html(请求的目标资源) HTTP/1.1(请求采用的协议和版本号)     
（2）Accept: */*(客户端能接收的资源类型)     
（3）Accept-Language: en-us(客户端接收的语言类型)     
（4）Connection: Keep-Alive(维护客户端和服务端的连接关系)     
（5）Host: localhost:8080(连接的目标主机域名和端口号)     
（6）Referer: http://localhost/links.asp(告诉服务器我来自于哪里)     
（7）User-Agent: Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/75.0.3770.100 Safari/537.36 （用户代理：简称UA。内容包含发出请求的用户信息，使得服务器能够识别客户端使用的操作系统及版本、CPU类型、浏览器及版本、浏览器渲染引擎、浏览器语言、插件等。）     
（8）Accept-Encoding: gzip, deflate(客户端能接收的压缩数据的类型) 
（9）If-Modified-Since: Tue, 11 Jul 2000 18:23:51 GMT(缓存时间)  
（10）Cookie(客户端暂存服务端的信息) 
（11）Date: Tue, 11 Jul 2000 18:23:51 GMT(客户端请求服务端的时间)

##### 响应(服务端->客户端[response]) 
（1）响应头：HTTP/1.1(响应采用的协议和版本号) 200(状态码) OK(描述信息)
（2）Location: http://www.baidu.com(服务端需要客户端访问的页面路径) 
（3）Server:apache tomcat(服务端的Web服务端名)
（4）Content-Encoding: gzip(服务端能够发送压缩编码类型) 
（5）Content-Length: 80(服务端发送的压缩数据的长度，在http1.1长连接里，如果Transfer-Encoding: chunked，则没有该头部，二选一) 
（6）Content-Language: zh-cn(服务端发送的语言类型) 
（7）Content-Type: text/html; charset=GB2312(服务端发送的类型及采用的编码方式)
（8）Last-Modified: Tue, 11 Jul 2000 18:23:51 GMT(服务端对该资源最后修改的时间)
（9）Refresh: 1;url=http://www.it315.org(服务端要求客户端1秒钟后，刷新，然后访问指定的页面路径)
（10）Content-Disposition: attachment; filename=aaa.zip(服务端要求客户端以下载文件的方式打开该文件)
（11）Transfer-Encoding: chunked(分块传递数据到客户端，如果有这个头部，则没有content-type头）  
（12）Set-Cookie: SS=Q0=5Lb_nQ; path=/search(服务端发送到客户端的暂存数据，可以设置为httponly，这样浏览器document.cookie取不到，避免篡改)
（13）Expires: -1//3种(服务端禁止客户端缓存页面数据)
（14）Cache-Control: no-cache(服务端禁止客户端缓存页面数据)  
（15）Pragma: no-cache(服务端禁止客户端缓存页面数据)   
（16）Connection: close(1.0)/(1.1)Keep-Alive(维护客户端和服务端的连接关系)  
（17）Date: Tue, 11 Jul 2000 18:23:51 GMT(服务端响应客户端的时间)
在服务器响应客户端的时候，带上Access-Control-Allow-Origin头信息，解决跨域的一种方法    

原文链接：https://blog.csdn.net/weixin_37861326/article/details/82216068

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

cookie存储在客户端，如果不设置过期时间，生命期为浏览器会话期间，关闭浏览器窗口（这种为会话cookie，会话cookie一般不存储在硬盘而是保存在内存里），如果设置过期时间，关闭后再打开浏览器这些cookie仍然有效直到超过设定的过期时间（cookie保存到硬盘上）。    
session存储在服务端，session的存在在某种程度上依赖于cookie（个人理解这种说法不完全正确，因为在使用session的客户端存储可以使用cookie，也可以不使用cookie，比如可以使用token字段，使用cookie的时候可以这么说）。

cookie又有httponly类型的cookie，一般由服务端直接设置，并设置httponly属性，这样的cookie在浏览器端使用document.cookie获取不到，避免了篡改cookie，不管是httponly类型的cookie，还是document.cookie设置的cookie，浏览器像服务器发送请求时都会通过Request Header的Cookie字段带给服务端（携带请求头Cookie是浏览器自己完成的，不需要前端干预）。

后端session没用过，也不知道怎么使用，不做介绍。

#### cookie是http4里面的，cookie有局限性： ####   
（1）每个域名最多只能有20个cookie；    
（2）cookie的长度不能超过4kb；    
（3）不安全：如果cookie被人拦掉了，那个人就可以获取到所有session信息。加密的话也不起什么作用，有些状态不可能保存在客户端；    
（4）setItem,getItem,removeItem,clear等方法，不像cookie需要前端开发者自己封装setCookie，getCookie；    
（5）但是cookie也是不可或缺的，cookie的作用是与服务器进行交互，作为http规范的一部分而存在的，而web Storage仅仅是为了在本地“存储”数据而生；    
sessionStorage、localStorage、cookie都是在浏览器端存储的数据，其中sessionStorage的概念很特别，引入了一个“浏览器窗口”的概念，sessionStorage是在同源的同窗口中，始终存在的数据，也就是说只要这个浏览器窗口没有关闭，即使刷新页面或进入同源另一个页面，数据仍然存在，关闭窗口后，sessionStorage就会被销毁，同时“独立”打开的不同窗口，即使是同一页面，sessionStorage对象也是不同的。

#### http5有了webStorage，包含localStorage和sessionStorage：####    
（1）localStorage和sessionStorage仅用于浏览器端存储，不参与服务器端的通信；    
（2）webStorage的存储大小都是5M；    
（3）sessionStorage不在不同的浏览器窗口中共享，localStorage在所有同源窗口中都是共享的，cookie也是在所有同源窗口中都是共享的；

#### sessionStorage、localStorage和cookie的区别 ####
1）相同点是都是保存在浏览器端、且同源的；    
2）cookie数据始终在同源的http请求中携带（即使不需要），即cookie在浏览器和服务器间来回传递，而sessionStorage和localStorage不会自动把数据发送给服务器，仅在本地保存。cookie数据还有路径（path）的概念，可以限制cookie只属于某个路径下；    
3）存储大小限制也不同，cookie数据不能超过4K，同时因为每次http请求都会携带cookie、所以cookie只适合保存很小的数据，如会话标识。sessionStorage和localStorage虽然也有存储大小的限制，但比cookie大得多，可以达到5M或更大；    
4）数据有效期不同，sessionStorage：仅在当前浏览器窗口关闭之前有效；localStorage：始终有效，窗口或浏览器关闭也一直保存，因此用作持久数据；cookie：只在设置的cookie过期时间之前有效，即使窗口关闭或浏览器关闭；    
5）作用域不同，sessionStorage不在不同的浏览器窗口中共享，即使是同一个页面；localstorage在所有同源窗口中都是共享的；cookie也是在所有同源窗口中都是共享的；    
6）web Storage支持事件通知机制，可以将数据更新的通知发送给监听者；    
7）web Storage的api接口使用更方便。

#### 本地存储和服务端存储 ####
1）数据既可以在浏览器本地存储，也可以在服务器端存储；    
2）浏览器可以保存一些数据，需要的时候直接从本地存取，sessionStorage、localStorage和cookie都是由浏览器存储在本地的数据；    
3）服务器端也可以保存所有用户的所有数据，但需要的时候浏览器要向服务器请求数据；    
4）服务器端可以保存用户的持久数据，如数据库和云存储将用户的大量数据保存在服务器端 ，服务器端也可以保存用户的临时会话数据，服务器端的session机制，如jsp的session对象，数据保存在服务器上；    
5）服务器和浏览器之间仅需传递session id即可，服务器根据session id找到对应用户的session对象，会话数据仅在一段时间内有效，这个时间就是server端设置的session有效期；    
6）服务器端保存所有的用户的数据，所以服务器端的开销较大，而浏览器端保存则把不同用户需要的数据分别保存在用户各自的浏览器中，浏览器端一般只用来存储小数据，而非服务可以存储大数据或小数据服务器存储数据安全一些，浏览器只适合存储一般数据

原文链接：https://blog.csdn.net/weixin_42614080/article/details/90706499


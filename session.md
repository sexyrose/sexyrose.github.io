# Session原理和Tomcat实现分析

这篇文章挖掘Session的原理和tomcat实现机制。    
由于HTTP是无状态的协议，客户程序每次都去web页面，都打开到web服务器的单独的连接，并且不维护客户的上下文信息。如果需要维护上下文信息，比如用户登录系统后，每次都能够知道操作的是此登录用户，而不是其他用户。对于这个问题，存在三种解决方案：cookie，url重写和隐藏表单域。

1、cookie

   cookie是一个服务器和客户端相结合的技术，服务器可以将会话ID发送到浏览器，浏览器将此cookie信息保存起来，后面再访问网页时，服务器又能够从浏览器中读到此会话ID，通过这种方式判断是否是同一用户。

```sh
 1 请求：  
 2 POST /ibsm/LoginAction.do HTTP/1.1  
 3 Accept: image/gif, image/x-xbitmap, image/jpeg, image/pjpeg, application/x-shockwave-flash, application/vnd.ms-excel, application/vnd.ms-powerpoint, application/msword, */*  
 4 Referer: http://192.168.1.20:8080/crm/  
 5 Accept-Language: zh-cn  
 6 Content-Type: application/x-www-form-urlencoded  
 7 UA-CPU: x86  
 8 Accept-Encoding: gzip, deflate  
 9 User-Agent: Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 5.2)  
10 Host: 192.168.1.20:8080  
11 Content-Length: 13  
12 Connection: Keep-Alive  
13 Cache-Control: no-cache  
14    
15 username=jack  
16    
17 响应：  
18 HTTP/1.1 200 OK  
19 Server: Apache-Coyote/1.1  
20 Set-Cookie: JSESSIONID=3267A671BFEAA147A2383B7E083D4G7E; Path=/crm  
21 Content-Type: text/html;charset=GBK  
22 Content-Length: 436  
23 Date: Sat, 10 June 2009 12:43:26 GMT
```

生成响应的时候，服务器向客户端发送cookie。cookie的属性是JSESSIONID，值是267A671BFEAA147A2383B7E083D4G7E。以后每次客户端请求时，都会附上此cookie，服务器端就可以读取到。

```sh
1 GET /ibsm/ApplicationFrame.frame HTTP/1.1  
2 Accept: image/gif, image/x-xbitmap, image/jpeg, image/pjpeg, application/x-shockwave-flash, application/vnd.ms-excel, application/vnd.ms-powerpoint, application/msword, */*  
3 Accept-Language: zh-cn  
4 UA-CPU: x86  
5 Accept-Encoding: gzip, deflate  
6 User-Agent: Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 5.2)  
7 Host: 192.168.1.20:8080  
8 Connection: Keep-Alive  
9 Cookie: JSESSIONID=267A671BFEAA147A2383B7E083D4G7E 
```

服务器端根据读取到的JSESSIONID，在一个map里面查找其对应的session对象，这个map的key是jsessionid的值，value是session对象。


2、URL重写

重写这种方式，客户端程序在每个URL的尾部自动添加一些额外数据，这些数据以表示这个会话，比如
http://192.168.1.20:8080/crm/getuserprofile.html;jsessionid=abc123。URL重写的额外数据是服务器自动添加的，那么服务器是怎么添加的呢？Tomcat在返回Response的时候，检查JSP页面中所有的URL，包括所有的链接，和 Form的Action属性，在这些URL后面加上“;jsessionid=xxxxxx”。 添加url后缀的代码片段如下：
org.apache.coyote.tomcat5.CoyoteResponse类的toEncoded()方法支持URL重写。   

```sh
1 StringBuffer sb = new StringBuffer(path);
2         if( sb.length() > 0 ) { // jsessionid can't be first.
3             sb.append(";jsessionid=");
4             sb.append(sessionId);
5         }
6         sb.append(anchor);
7         sb.append(query);
8         return (sb.toString());
```

从上面URL的实现原理可知，URL重写有一个缺点：在你的站点上不能有任何静态的HTML页面(至少静态页面中不能有任何链接到站点动态页面的链接)。因此，每个页面都必须使用servlet或 JSP动态生成。即使所有的页面都动态生成，如果用户离开了会话并通过书签或链接再次回来，会话的信息都会丢失，因为存储下来的链接含有错误的标识信息－ 该URL后面的SESSION ID已经过期了。

3、隐藏表单域

这种方式借助html表单中的hidden来实现，适用特定的一个流程，但是不适用于通常意义的会话跟踪。

综上所述，session实现会话跟踪通常是cookie和url重写，如果浏览器不禁止cookie的话，tomcat优先使用cookie实现。

**服务器端实现原理**

Session在服务器端具体是怎么实现的呢？我们使用session的时候一般都是这么使用的：

request.getSession()或者request.getSession(true)。

这个时候，服务器就检查是不是已经存在对应的Session对象，见HttpRequestBase类
doGetSession(boolean create)方法：

```java
 1  if ((session != null) && !session.isValid())
 2             session = null;
 3         if (session != null)
 4             return (session.getSession());
 5 
 6 
 7         // Return the requested session if it exists and is valid
 8         Manager manager = null;
 9         if (context != null)
10             manager = context.getManager();
11         if (manager == null)
12             return (null);      // Sessions are not supported
13         if (requestedSessionId != null) {
14             try {
15                 session = manager.findSession(requestedSessionId);
16             } catch (IOException e) {
17                 session = null;
18             }
19             if ((session != null) && !session.isValid())
20                 session = null;
21             if (session != null) {
22                 return (session.getSession());
23             }
24         }
```

requestSessionId从哪里来呢？这个肯定是通过Session实现机制的cookie或URL重写来设置的。见HttpProcessor类中的parseHeaders(SocketInputStream input)：

```java
 1 for (int i = 0; i < cookies.length; i++) {
 2                     if (cookies[i].getName().equals
 3                         (Globals.SESSION_COOKIE_NAME)) {
 4                         // Override anything requested in the URL
 5                         if (!request.isRequestedSessionIdFromCookie()) {
 6                             // Accept only the first session id cookie
 7                             request.setRequestedSessionId
 8                                 (cookies[i].getValue());
 9                             request.setRequestedSessionCookie(true);
10                             request.setRequestedSessionURL(false);
11                             
12                         }
13                     }
14 }
```

或者HttpOrocessor类中的parseRequest(SocketInputStream input, OutputStream output)

```java
1 // Parse any requested session ID out of the request URI
2         int semicolon = uri.indexOf(match);  //match 是";jsessionid="字符串
3         if (semicolon >= 0) {
4             String rest = uri.substring(semicolon + match.length());
5             int semicolon2 = rest.indexOf(';');
6             if (semicolon2 >= 0) {
7                 request.setRequestedSessionId(rest.substring(0, semicolon2));
8                 rest = rest.substring(semicolon2);
9             } else {
10                 request.setRequestedSessionId(rest);
11                 rest = "";
12             }
13             request.setRequestedSessionURL(true);
14             uri = uri.substring(0, semicolon) + rest;
15             if (debug >= 1)
16                 log(" Requested URL session id is " +
17                     ((HttpServletRequest) request.getRequest())
18                     .getRequestedSessionId());
19         } else {
20             request.setRequestedSessionId(null);
21             request.setRequestedSessionURL(false);
22         }
```

里面的manager.findSession(requestSessionId)用于查找此会话ID对应的session对象。Tomcat实现
是通过一个HashMap实现，见ManagerBase.java的findSession(String id):

```java
1         if (id == null)
2             return (null);
3         synchronized (sessions) {
4             Session session = (Session) sessions.get(id);
5             return (session);
6         }
```

Session本身也是实现为一个HashMap，因为Session设计为存放key-value键值对，Tomcat里面Session实现类是StandardSession，里面一个attributes属性：

```java
1     /**
2      * The collection of user data attributes associated with this Session.
3      */
4     private HashMap attributes = new HashMap();
```

所有会话信息的存取都是通过这个属性来实现的。Session会话信息不会一直在服务器端保存，超过一定的时间期限就会被删除，这个时间期限可以在web.xml中进行设置，不设置的话会有一个默认值，Tomcat的默认值是60。那么服务器端是怎么判断会话过期的呢？原理服务器会启动一个线程，一直查询所有的Session对象，检查不活动的时间是否超过设定值，如果超过就将其删除。见StandardManager类，它实现了Runnable接口，里面的run方法如下：

```java
 1     /**
 2      * The background thread that checks for session timeouts and shutdown.
 3      */
 4     public void run() {
 5 
 6         // Loop until the termination semaphore is set
 7         while (!threadDone) {
 8             threadSleep();
 9             processExpires();
10         }
11 
12     }
13 
14     /**
15      * Invalidate all sessions that have expired.
16      */
17     private void processExpires() {
18 
19         long timeNow = System.currentTimeMillis();
20         Session sessions[] = findSessions();
21 
22         for (int i = 0; i < sessions.length; i++) {
23             StandardSession session = (StandardSession) sessions[i];
24             if (!session.isValid())
25                 continue;
26             int maxInactiveInterval = session.getMaxInactiveInterval();
27             if (maxInactiveInterval < 0)
28                 continue;
29             int timeIdle = // Truncate, do not round up
30                 (int) ((timeNow - session.getLastUsedTime()) / 1000L);
31             if (timeIdle >= maxInactiveInterval) {
32                 try {
33                     expiredSessions++;
34                     session.expire();
35                 } catch (Throwable t) {
36                     log(sm.getString("standardManager.expireException"), t);
37                 }
38             }
39         }
40 
41     }
```

Session信息在create,expire等事情的时候都会触发相应的Listener事件，从而可以对session信息进行监控，这些Listener只需要继承HttpSessionListener，并配置在web.xml文件中。如下是一个监控在线会话数的Listerner：

```java
import java.util.HashSet;

import javax.servlet.ServletContext;

import javax.servlet.http.HttpSession;

import javax.servlet.http.HttpSessionEvent;

import javax.servlet.http.HttpSessionListener; 

public class MySessionListener implements HttpSessionListener {       

 public void sessionCreated(HttpSessionEvent event) {             

 HttpSession session = event.getSession();             

ServletContext application = session.getServletContext();                           

 // 在application范围由一个HashSet集保存所有的session             

 HashSet sessions = (HashSet) application.getAttribute("sessions");             

if (sessions == null) {                    

sessions = new HashSet();                    

application.setAttribute("sessions", sessions);             

}                           

// 新创建的session均添加到HashSet集中             

 sessions.add(session);             

// 可以在别处从application范围中取出sessions集合             

// 然后使用sessions.size()获取当前活动的session数，即为“在线人数”      

}       

public void sessionDestroyed(HttpSessionEvent event) {             

HttpSession session = event.getSession();             

 ServletContext application = session.getServletContext();             

 HashSet sessions = (HashSet) application.getAttribute("sessions");                           

 // 销毁的session均从HashSet集中移除             

sessions.remove(session);      

}

}
```

---
title: Let's Talk About Web Server
date: 2022-04-13 00:16:32
tags: Web Server
categories: Java
---

Apache, Tomcat, Jetty, Spring, SparkJava. When a beginner starts a backend application, frameworks take care everything, we don't need to care about how it works just focus on the API programming which is good and bad. I want to talk about them from my understanding, my learning path.

### What is a server
When I start my work for SiriusXM, I did a lot of network IO work involved lots of TCP connections (Client/Server). At that time, I don't even touch HTTP server. Server is not only meaning web/HTTP server. For me, the server means a pure TCP server, it can be database server, or websocket server or any computer you can get a response.

```
    sock = socket(AF_INET,SOCK_STREAM,0);
    int ret = bind(sock, port); // bind port
    ret = listen(sock,1); // listen
    connfd = accept(sock, client, client_addrlength); // accept connection
    int fd = open("hello.html",O_RDONLY);
    sendfile(connfd,fd,NULL,2500); // send response
```
This is a simple TCP server which does not handle HTTP and just return a HTML file. However when you connect to this server through a browser, it will still get HTML file back and see the HTML content. So the core of the web HTTP server is just a TCP server return a HTML file.

### Apache
The HTTP is a protocal which define the rules how client and server communicate with each other and transfer data, like only client can fire a connection to server then server will give response back, etc. We often compared it with websocket. HTTP can be not only used in web browser, it can be used any application. HTTP connection can be considered 3 steps. First is to establish a TCP connection. Second client generates HTTP format request and send to server. Last server sends back HTTP format response to client.

Apache is a widely used HTTP server. Apache has a process of URI Translation, it will translate the URL to find which resource it looking for and return the file, such as HTML, tar, etc. It kinda looks like what we did before, return a file but the pure TCP server does not have ability to analysis URL, the server can only accept the connection with the hostname and port. What's more, apache can analysis more based on HTTP protocal.

It can only provide static content. For a web serverm, all web page needs to be written and placed in the designate path.
```
                request
web broswer     ------>       HTTP server
                               get file
                <------
                response
```
_When we hit `http://laravel.com`, Apache analyse the request information knows that we didn't specify a file, so it looks for a directory index and finds `index.php`. `.php` file should send to PHP app. Then Apache receives the output from PHP and sends it back over the Internet to a user's web browser._


### Tomcat/Jetty
Static content is not enough for complex web service, and we need dynamic content. Also when we have complicated web service and we don't want to couple web service with HTTP server. HTTP server should not be responsible for finding the right service to ask for content. As a result, we define a abstract layer, which are interfaces for Java application, Servlet.
```
               request
web broswer    ------>     HTTP server   ----->    Servlet   ---->   service
               <------                   <-----              <----
               response
```
This is not completely right, it is easy to understand. Servelets need to be deployed in Servlet container. Servlet container will find right servlet for the request.
```
public interface Servlet {
    void init(ServletConfig config) throws ServletException;
    ServletConfig getServletConfig();
    void service(ServletRequest req, ServletResponse res）throws ServletException, IOException;
    String getServletInfo();
    void destroy();
}
```
For a simple example, we can inherit HttpServlet and implement service we want and map and request with corresponding services in web.xml. Then we deploy these into web container. Tomcat and Jetty is a web container (HTTP server plus servlet container).

In constructing response, we can put data from dababase in to it.
```
protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
    PrintWriter out = response.getWriter();
    response.setContentType("text/html;charset=utf-8");
    out.println("<HTML>My Servlet!");
    out.println(data);
    out.println("</HTML>");
}
```
For now, all frontend pages are generated from backend and sent to web browser. It is heavy load. Nowadays, we separate frontend and backend. Frontend like Angular/React call backend endpoint to get data. Backend only return json instead of the whole HTML page.

_Like PHP/ASP, Java provides JSP to generate HTML from backend. Developer can write Java code in HTML style template and generate HTML file. JSP is a wrapped servlet, we don't talk too much here._

### Other Framework
Spring wraps tomcat and SparkJava uses Jetty. SpringMVC is based on servlet. If we don't know what is servlet and how does it work, it is hard to understand [SpringMVC](https://stackify.com/spring-mvc/). If there is anything wrong here, please let me know by email.


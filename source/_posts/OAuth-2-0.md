---
title: OAuth 2.0
date: 2018-11-15 19:00:32
tags: OAuth
categories: Protocol
---

Many Apps support third party login, like Facebook, Wechat, Dribbble etc. When we develop an app that need to support this feature, we need to know about OAuth protocol.

>OAuth official website
>OAuth 2.0 is the industry-standard protocol for authorization. OAuth 2.0 focuses on client developer simplicity while providing specific authorization flows for web applications, desktop application, mobile phones and living room devices.

Why we need OAuth protocol?
Usually, if we want to get protected resource, the client needs to send user authorization certificate to server, such as username and password. For example, if user login official facebook or dribbble client to get shots, it is safe to send username and password. What if the third app want to get the these protected shots?

> Thrid app -> Server : Can I get protected resource?
> Server -> Third app : Sure. I need username and password first.
> Third app -> User : Can I get your username and password?
> User -> Third add : No!!!
> Third app : Whatever, I will do it myself.

So let's use a dribbble shots api to get picture shots. In chrome, we can make a simple get request
> GET
> https://api.dribbble.com/v1/shots

What we get is
```json
{
    "message": "Bad credentials."
}
```
Just like other services, the server needs user cerdential in order to fetch protected data. That is where OAuth comes into play.

OAuth protocol define four main characters:
1. Resource owner: the owner of the resource, usually called end-user.
2. Resource server: the server of the protected data, client can get data based on access Token.
3. Client: represent user to get protected resource, any application.
4. Authorization server: authorize client access Token.

OAuth 2.0 process can be described by the following diagram.
![](https://st.deepzz.com/blog/img/oauth2-roles.jpg)

- Client send request to resource owner for credential authorization. This request can be sent by authorization server.
- If the end-user authorize the request, the application receives an authorization grant. This will represent resource owner.
- The application requests an access token from the authorization server.
- If the application identity is authenticated and the authorization grant is valid, the authorization server issues an access Token to the application.
- Client/Application request the resource from resource server by access Token.
- Check the access Token and send protected resource to the client.

This flow is the general process of OAuth 2.0 protocol. Let's do it with Dribbble example. Here is the details of instruction `Authorization Code Mode`: http://developer.dribbble.com/v1/oauth/

First, if a client wants to get authroized, it must be registered with the service. You need to define a callback URL which is the service will redirect the user after they authorize the request. The service will provide credentails which are client ID (public) and client secret (private).

1. Client direct user to Authorization Server with client ID, scope, redirect uri.
2. Authorization server check the validation of the client and ask user authentication.
3. If user authorize the request, authorization server will redirect user-agent to callback uri long with an authorization code.
4. After Client gets code, send code, client ID and client secret to authorization server to get access Token. (Server will valid client and also code.)
5. Client use access Token to ask for protected resource.

Here is the Dribbble authorization address.
```
GET https://dribbble.com/oauth/authorize
```
And we create a GET request
```java
private static String getAuthorizeUrl() {
    String url = Uri.parse(URI_AUTHORIZE)
                    .buildUpon()
                    .appendQueryParameter(KEY_CLIENT_ID, CLIENT_ID)
                    .build()
                    .toString();
    url += "&" + KEY_REDIRECT_URL + "=" + REDIRECT_URI;
    url += "&" + KEY_SCOPE + "=" + SCOPE;
    return url;
}
```
The reason why I use `+=` to add url and scope because when we use `appendQueryParameter` it will add some character into our url, something like
```
redirect_uri=http://www.dribbbo.com&scope=public+write
redirect_uri=http%3A%2F%2Fwww.dribbbo.com&scope=public%2Bwrite
```

You can try this url directly in your web browser. The address will immediatelly change from A to B.
```
A: https://dribbble.com/oauth/authorize?client_id=<client_id>&redirect_url=<redirect_url>&scope=<scope>
B: http://www.dribbbo.com/?code=<code>
```

In the Authorization activity, we set a webView and load this url. As the url will get changed, we can use `shouldOverrideUrlLoading` to catch new url and get the code.
```java
webView.setWebViewClient(new WebViewClient(){
    @Override
    public boolean shouldOverrideUrlLoading(WebView view, String url) {
        if(url.startsWith(Auth.REDIRECT_URI)) {
            Uri uri = Uri.parse(url);
            Intent intent = new Intent();
            intent.putExtra(KEY_CODE, uri.getQueryParameter(KEY_CODE));
            setResult(Activity.RESULT_OK, intent);
            finish();
        }
        return super.shouldOverrideUrlLoading(view, url);
    }
});
```
Now we get authorization code. Next step is to fetch access token. Exchange address is
```
POST https://dribbble.com/oauth/token
with client_id, client_secret, code
```
Here is the code to send post request to get access token.
```java
OkHttpClient client = new OkHttpClient();
RequestBody requestBody = new FormBody.Builder()
                                        .add(KEY_CLIENT_ID, CLIENT_ID)
                                        .add(KEY_CLIENT_SECRET, CLIENT_SECRET)
                                        .add(KEY_CODE, authCode)
                                        .add(KEY_REDIRECT_URL, REDIRECT_URI)
                                        .build();
Request request = new Request.Builder()
                            .url(URI_TOKEN)
                            .post(requestBody)
                            .build();
Response response = client.newCall(request).execute();
```
You can try curl to get Dribbble response
```
curl --data "client_id=<ID>&client_secret=<SECRET>&code=<CODE>&redirect_uri=<CALLBACKURL>" https://dribbble.com/oauth/token
```
```json
{
    "access_token" : "********",
    "token_type" : "bearer",
    "scope" : "public write"
}
```
We can create a JSONObject to get access token. After we get the token, we can use this token along with API to ask for data.
```
curl -H "Authorization: Bearer <ACCESS_TOKEN>" https://api.dribbble.com/v2/user
```
This is basically how OAuth 2.0 Authorization Code works. When I first time tried to understand this process, I wonder why they can not just return access token first time. Why the protocol needs to send code first, then exchange for token?

Here we should bring up difference between user-agent and client.
- User-agent is like front-end (browser), user use it to communicate with server.
- Client is the part asks to access the protected resource.

So when authorization server transmits of the access token directly to the user-agent, usually a browser, Potentially exposing it to others, including the resource owner (As I said above to try the get code request in browser). However, in authorization code mode, it sends code to user-agent, server will get it and next steps will be a server-to-server process. Access token and client secret will never get exposed. These applications have the decoupled user-agent and client (separate ui and server).

If the user-agent and the client are coupled, like in-browser javascript application or native mobile app. It won't need this code to get access. The token and client secret will still be shared with resource owner. The code and client secret just make the flow more complex without adding any more real security. That was when I implemented facebook login in my [Shopping app][1], the [facebook android developer][2] tutorial did not give me the instructions for authorization code and client secret. It only provides token to make graph request. This process directly returning access token is called implicit authorization.

[1]:https://github.com/bbbbyang/Shopping
[2]:https://developers.facebook.com/docs/facebook-login/android/



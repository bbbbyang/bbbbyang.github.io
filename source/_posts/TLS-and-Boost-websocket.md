---
title: TLS and Boost websocket
date: 2019-09-28 20:38:12
tags: TLS
categories: Protocol
---

### TLS Process
My current project is dealing with boost websocket and tls protocol. Currently, our company use boost websocket(ws) to build connection between client and server. Client will send frame registration/request to server and server will send result/frame back. When I took it over, the websocket library I our company does not implement tls verification for ws and I need to finish this feature.

No matter what protocol websocket or http we use for communication, we send message through the network. Without an encryption, all the messages are plaintext. A man in middle can hack the message and get the information, credit card info, account and password, etc.

```
   Client                                     Server
      |           username password             |
      |          ------------------->           |
      |                                         |
      |                                         |
```
We definitely do not want to send plaintext through the network. Man in the middle(MITM) can attack easily. Any one in the network can see this message. So the first thought is to encrypt the message. 

For the encryption, symmetric cryptographic algorithm provides one key. Both encryption and decryption use the same key.

```
F(key, data) = ciphertext
F'(key, ciphertext) = data
```

```
   Client                                     Server
      |PK         OIUA(W&*@!NA<>SDH             | PK
      |          ------------------->           |
      |                                         |
      |                                         |
```

The most widely used algorithm of symmetric encryption is called AES. The pros of this is that it is fast and it is good to encrypted large data. However, it is hard to determine the key between client and server. Whether client send key to server, or server send the key to client. MITM can get the key and hack the connection. Encryption will become meaningless.

Here we have asymmetric encryption. For asymmetric encryption, it will generate two keys, one is called public key, another one is called private key. Any info encrypted by one of them can only get decrypted by the another key.

```
F(Pk1, data) = ciphertext
F'(Pk2, ciphertext) = data
F(Pk2, data) = ciphertext
F'(Pk1, ciphertext) = data
```
Now the message between client and server will be like this
```
   Client                                     Server
      |Public key      OIUA(W&*@!NA<>SDH        | Private key
      |              ------------------->       |
      |               UIA*(&&*QJKASDKKAS        |
      |              <-------------------       |
```
On client side, we use the public key to encrypt the message then send it to server. Private key will only kept by server. The message which is encrypted by the public key can only decrypted by private key. If MITM intercepts the message, he does not have the private key, so he can not decrypted message. It seems to work fine. There are problems. 

- the process of asymmetric encryption and decryption are slow.

What's more, there are two security problem

- MITM can have the public key and is able to decrypt the message from server to client
- It can not guarantee that the public key for client is from the server not MITM

Let's solve this one by one. For the disadvantage of asymmetric encryption and public key exposure we can combine the symmetric encryption and asymmetric encryption.

1. After connection established, client will ask for public key
2. Server sends public key to the client
3. Client generates a random number P encrypted by public key and sends it to the server
4. Server will decrypt the number P and confirm the P will be the key for symmetric encryption
5. The following messages will be symmetric encrypted by this number P

```
   Client                                     Server
      |               ask for public key        | ---
      |              ------------------->       |   |
      |               send public key           |   |
      |              <-------------------       |   | --> asymmetric encryption
      |                send num P as key        |   |
      |              ------------------->       |   |
      |   confirm P as symmetric encryption key |   |
      |              <-------------------       | ---
      |PK              OIUA(W&*@!NA<>SDH        | PK
      |               ------------------->      |
```
So Even though MITM gets the message he does not know the newly generated key for symmetric encryption by the client. He can not decrypt the message. But the second problem still not get fixed. How about the MITM get the very first message asking for public key after the connection established.
```
   Client                    MITM                   Server
      |  ask for public key   |  ask for public key   |
      | ------------------->  | ------------------->  |
      |  send fake public key |  send public key      |
      |  <------------------- | <-------------------  | 
      |  send num P as key    |  send num P as key    |
      |  -------------------> | ------------------->  |
      | confirm P as the key  | confirm P as the key  |
      | <-------------------  | <-------------------  |
```
If the MITM get the first message and send a fake public key to trick the client he is the server. Now we have a license to solve this problem. It is called Certification Authority(CA).

In the whole internet, all users will admit some authorized institutions, like Microsoft, Google. They will be considered as Root Authority so every browser will be embedded with these institutions' public key. You won't have any authority issue with these public keys when you install your browser.

When server apply for a license with all company information, the institution will review the information. After the review, institution will generate two keys, one is public key another is private key and a CA license. Private key will kept by the server.

CA license has all company info and the server public key. And hash algorithm will be used to generate a hash value which is called digest. Institution will use its private key to encrypt the digest as a signature. The signature will append to the end of the CA.

The process will be

1. Instead of asking for public key after connection, client will ask for CA license.
2. Server sends CA license to the client
3. Client will check the CA and find the Root institution public key from embedded keys in browser.
4. Use the Root authority public key decrypt signature to get the digest
5. Do hash calculation of CA info to get a digest
6. Compare these two digests (hash value), if they match the public key in CA is authorized

Even if MITM changed the license, he does not have the Root authority private key to encrypt the CA. The client will know it. So for now, we have solved the potential security problems. The entire process will be 

1. Client -> Server. SSL version, asymmetric encryption algorithm, random number 1 (Client Hello)
2. Server -> Client. Confirm SSL version, symmetric encryption algorithm, random number 2, CA license (Server Hello, Server Certificate, Server Hello Done)
3. Client checks CA license
4. Client -> Server. Pre-master number (Client Key exchange, Client Certificate*)
5. Client, Server use three numbers to generate symmetric encryption key
6. Client -> Server. Confirm symmetric encryption key and test message (Change Cipher Spec, Encrypted Handshake Message)
7. Server -> Client. Confirm symmetric encryption key and test message (Change Cipher Spec, Encrypted Handshake Message)

### Boost Library
We use boost.beast library and follow [async websocket example][1] 1_66_0 version to build our own websocket library. And I also found a very simple [server echo example][2] used boost.beast. I played with this code to have a better understanding of beast library.

When you are trying to use this server echo example, there are some compile error need to be fixed.

initialization of io_context should be updated like
```java
boost::asio::ssl::context ctx(boost::asio::ssl::context::sslv23);

```
and also for dh512.pem is small. Should use dh1024.pem instead.

When I take over this work, the current websocket we built not implemented SSL. This is not hard to implement. Follow the async example in the boost.beast website and load the key and certificate will be fine. However there are two problems I want to address. I spent lots of time to figure them out.

First, some messages I sent to server are broken. Before I implemented SSL, it worked fine. Then I add TLS on websocket, and made server keep sending same messages to client. One or two messages can not get parsed by protobuf. 

```c++
sendMessage( const std::string& serializedMessage )
{
    ...
    _webSocket.async_write(
            boost::asio::buffer( serializedMessage ),
            std::bind(
                    &Session::onWrite,
                    this->shared_from_this(),
                    std::placeholders::_1,
                    std::placeholders::_2 ) );
}
```
I did not control the lifetime of the `serializedMessage` string parameter extends until the `async_write` operation is complete, which means `async_write` was still waiting while `sendMessage` function already finished. `serializedMessage` will be released but `async_write` was not completed. You can check the [issue][3] I raised in Github, the author gave me a explanation in detail.

The second problem is that websocket reconnection. In our requirements, if the connection is not established, client should retry connect every few seconds. We followed the 1_66_0 boost.beast websocket example. 
```c++
std::make_shared<session>(ioc)->run(host, port, text);
```
The old solution is that we have a shared pointer pointing to the session, so every time we want to reconnect we reset the pointer and make a new session to run again. This seems works fine. After I let my client running all night trying to reconnect every few seconds. The client crashed several times every around 6 hours. Resource runs out exception. I used `pgrep` and `ps -T -p` to check the task. It created so many client session tasks.

The reason is that it is a shared pointer and in session it has operations like
```c++
ws_.async_handshake(host_, "/",
    std::bind(
        &session::on_handshake,
        shared_from_this(),
        std::placeholders::_1));
```
The client session will point to itself. Even though you reset the shared pointer, the dead client session task will not get released.

So for reconnection, `asio::ssl::stream` can be used again after being closed. Instead of recreating client session, you need to close the dead ssl connection and create a new `boost::asio::ip::tcp::socket` each time you reconnect.

```
_ws->lowest_layer().close();
_ws.reset(new WebSocket(_ioContext, _sslContext));
```
This is not enough, the io_context will get returned if all the handler returned. Even though you have a new recreation. There are two solutions I listed [here][4]. You can either set a work guide for the io_conetxt or use a callback timer inside the websocket connection so that the handler can not finish working.

[1]:https://www.boost.org/doc/libs/1_66_0/libs/beast/doc/html/beast/examples.html
[2]:https://github.com/AdamMagaluk/asio-ssl-mutual-auth
[3]:https://github.com/boostorg/beast/issues/1707
[4]:https://github.com/boostorg/beast/issues/1747
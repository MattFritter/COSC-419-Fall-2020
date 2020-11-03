# COSC 419: Topics in Computer Science
# Fall 2020 - Lab 7

In this lab, we'll be exploring ways of securing our web application against XSS and SQLi attacks and hiding server information using HTTP headers.

## Table of Contents
- [Server Camouflage](#camouflage)
- [Client-Side XSS Protection](#client-side)
- [Server-Side XSS & SQLi Protection](#server-side)

<a name="camouflage"></a>
## Camouflaging our Server (6 marks)

Presently, our server is returning detailed information about our web stack, including the Apache and PHP versions currently used, and the underlying operating system. Giving away this data makes it easier for bad actors to identify potential attacks based on version-specific and software-specific vulnerabilities.

You can view the HTTP headers that your server returns using the Inspect tool in your browser, then selecting the "Network" tab. Click your IP resource and then select the 'Headers' subtab to show your Response Headers. Alternatively, you can use the <a href="http://www.rexswain.com/httpview.html">Rex Swain's HTTP Viewer</a>. Your headers should look similar to the following image:

<img src="https://i.imgur.com/fJcP2hC.png" width="80%"/>

Specifically, we want to eliminate the Server and X-Powered-By headers, as they return detailed information about our system. We'll begin with the X-Powered-By header. This header is added by PHP itself, and is configurable through your ```/etc/php.ini``` file.

Open this file in a text editor of your choice, and find the line that says ```expose_php = On```, and edit it to ```expose_php = Off```. Save and close the file, then restart the server using ```service httpd restart```. When the server comes back online, check your HTTP headers again. The X-Powered-By header should no longer be included in the response. If this doesn't work, you may need to give your server a hard restart to get it to re-read the PHP.ini file. You can do this with ```reboot```, but you'll need to reconnect afterwards. If you do reboot your server, double check that SELINUX is set to ```permissive``` mode when it comes back (see lab 1 document for how to do this).

The Server HTTP header is more difficult to deal with. By default, Apache will not let you remove it or overwrite it completely - it will only let you remove detailed version and system information, but continue to display that it is an Apache-based server. In order to change it's value, we'll need to install the ```mod_security``` package. Use the following commands to install the package:

	yum install mod_security
	service httpd restart

Note that the mod_security package is an all-in-one security package that can do a lot more than remove and edit HTTP headers. We'll be using it in the next lab to further secure our web server. For now though, we can skip most of the configuration.

Open ```/etc/httpd/conf.d/mod_security.conf``` in a text editor of your choice, and add the line ```SecServerSignature "Your Text Here"``` to the top, just inside the ```<IfModule mod_security2.c>``` statement. This will replace the Server header contents with the specified text. You may use any text you want, but a relatively common tactic is to use a signature from an unrelated server, such as ```Microsoft-IIS/5.0```. Close and save the file, then restart Apache.

<img src="https://i.imgur.com/iwVSkzE.png" width="60%">

Try opening your web browser inspector (or use Rex's site), and look at your HTTP headers again. You might notice that you've successfully replaced the ```Apache``` token, but the tokens for ```mod_wsgi``` and ```Python``` are probably still kicking around, even though we're currently serving a PHP website. To fix this and remove those tokens entirely, open up your main Apache config file at ```/etc/httpd/conf/httpd.conf```. Add a line new the top that looks like this:

<img src="https://i.imgur.com/dmLgqvw.png" width="60%"/>

Restart Apache. If you've followed these steps, your HTTP headers should now look like this. Note that we are no longer exposing any information about internal software components or versions, and we've disguised our server as a Microsoft IIS server instead of Apache:

<img src="https://i.imgur.com/MemCYWt.png" width="80%"/>

While this might trip up bots that merely deduce the server architecture from HTTP headers, there are many ways to fingerprint a web server. Use <a href="https://w3dt.net/tools/httprecon">HTTPRecon</a> to perform a fingerprinting analysis on your server (enter the IP, *without* http://). You'll most likely get results that look like this:

<img src="https://i.imgur.com/TWdaknf.png" width="80%"/>

D'oh! While changing the Server header and hiding the X-Powered-By header gives the outward appearance of a different server returning responses, things like the ordering of the HTTP headers, the precise spelling/capitalization of header values, and content returned from default error pages (404, 403, 500, etc) all leak some information about what server we're running. This allows fingerprinting services like HTTPRecon to identify that we are using Apache.

It is nearly impossible to completely remove all traces of Apache from our wep responses, but we can make our facade a little stronger by understanding the server we're trying to camoflauge ourselves as. First, we'll edit our ```/etc/httpd/conf.d/mod_security.conf``` again, changing the Server banner to ```Microsoft-IIS/6.0```, as this is the more recent version of IIS. We can aid the subterfuge by adding our own X-Powered-By banner, but making it an ASP.NET banner, which is a more likely back-end language for IIS than PHP. 

First, open ```/etc/httpd/conf/httpd.conf``` in the text editor of your choice. Then, scroll all the way to the bottom of the file, and add a new line:

	Header always set X-Powered-By ASP.NET
	
This will always attach an X-Powered-By header to responses, with a value of "ASP.NET" - an expected response header from a stock IIS server. Then, add two more headers to your response: a header named ```X-AspNet-Version```, with a value of ```4.0.30319``` and a header named ```X-AspNetMvc-Version``` with a value of ```5.2```. 

This disables the server signature (in our case, Microsoft IIS, but by default, Apache) from being displayed at the bottom of standard Apache error pages (404, 403, 500, etc). Restart your Apache server when you're done.

Now, if we rerun the HTTPRecon fingerprint test, we should hopefully see:

<img src="https://i.imgur.com/sTB0ACi.png" width="80%"/>

Which means that we've successfully fooled HTTPRecon into believing our server is an IIS web server. However, note that different fingerprinting will obtain different results. For example, the <a href="http://www.net-square.com/httprint.html">Httprint</a> fingerprinting software still identifies the server as an Apache webserver.

Remember that obscurity is not security. Disguising our web server can help prevent some attacks, depending on the attackers fingerprinting technique, but we shouldn't consider it a "security" measure. What we're doing is deflecting low-effort attackers, allowing us to focus on more serious security concerns. You can find out more about fingerprinting at the OWASP website <a href="https://www.owasp.org/index.php/Fingerprint_Web_Server_(OTG-INFO-002)">here</a>.

<a name="client-side"></a>
## Client Side XSS Protection (4 marks)

In order to protect our users against potential XSS attacks, we'll go ahead and implement a Content Security Policy (CSP), and enable XSS protection using HTTP headers.

First, open up your Apache config file (```/etc/httpd/conf/httpd.conf```) again, and go to the bottom of the file. Then, we'll add a new header for our Content Security Policy:

	Header always set Content-Security-Policy "default-src 'self'"

If you go to your homepage now, you'll probably notice that this has completely broken the styling of your page. This is because by default, the client will now only allow resources to be loaded from our own domain (our IP) - we can't load BootStrap or fonts from our CDNs. If you open your browser console, you'll see the error messages generated by these failures:

<img src="https://i.imgur.com/PmP4nbZ.png" width="80%"/>

Use these errors as a guide for how to write your CSP, such that you allow resources from your CDNs and Google Fonts (assumed to be safe), but block other sources. To break it down:

1. You'll need to approve 'self', 'unsafe-inline', stackpath.boostrapcdn.com, and fonts.googleapis.com as sources in your style-src
2. You'll need to approve 'self', cdnjs.cloudflare.com, code.jquery.com, and stackpath.bootstrapcdn.com as sources for script-src
3. You'll need to approve 'self', 'unsafe-inline', and fonts.gstatic.com as sources for font-src

The format for a Content Security Policy is as follows:

- Single quotes are used to wrap the sources that are *not* websites, such as 'self' and 'unsafe-inline'
- Semicolons are used to seperate lists of sources, such as ```default-src 'self'*;* style-src 'self' 'unsafe-inline'```
- Sources are separated by a space
- Web domains do *not* get wrapped in single quotes, and the asterisk character can be used as a wildcard: ```default-src 'self' *.google.com;```

Here's an example with some domain sources included to get you started:

	Header always set Content-Security-Policy "default-src 'self'; style-src 'self' 'unsafe-inline' stackpath.bootstrapcdn.com; script-src 'self' cdnjs.cloudflare.com;"
	
Once you think you're CSP is complete, save and close it, and restart your Apache server. If this was successful, your homepage should load normally, and you shouldn't have any errors in your browser console logs.

As a final step, we'll enable the client-side XSS filtering. Although this is usually unnecessary, it can't hurt, and could provide additional protection against reflected XSS attacks.

Add one more HTTP header with a name of ```X-XSS-Protection``` and a value of ```1, mode=block```. This will automatically block pages which the browser detects may contain a reflected XSS error. Restart Apache to enable your changes.

If you want to read more about Content Security Policies, the Mozilla Developer Network (MDN) documentation is excellent, available <a href="https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP">here</a>.

<a name="server-side"></a>
## Server-Side USer Input Validation (4 marks)

Now, let's go ahead and do some server-side validation of data to provide some **real** security. We'll do this using Laravel's ```validate()``` function in some new middleware that we'll create. To begin, we can create a new middleware file with PHP artisan:

	php artisan make:middleware validateRoomName

This will create a new middleware file in your Laravel ```app/Http/Middleware``` folder. Go ahead and open the new ```validateRoomName.php``` file. Inside, you can define a new middleware function inside ```handle()```. Your middleware file should be applioed to the POST route for the ```/create``` URI. This middleware should validate the incoming request from the client, which contains the ```roomName``` POST parameter specifying the name for a new chatroom.

Using the lecture slides from last lecture, combined with the documentation on available validation types in <a href="https://laravel.com/docs/8.x/validation#available-validation-rules">Laravel available here</a>, create a validation that requires that the room name must be:

1. Required
2. a String type
3. a max length of 100 characters
4. a min length of 5 characters

Remember, you can use the pipe character, ```|```, to combine multiple validation rules together. To help you get started, here's an example of a simple validation middleware that checks ```myVar```, making it ```required``` and also having a max length of 10. Note that we can rely on the ```validate()``` function to redirect the user for us if the validation fails - we don't have to actually do anything with it:

<img src="https://i.imgur.com/JTAQs7l.png" width="60%"/>

Don't forget to register your middleware in your ```Kernel.php``` file, following the directions from the lecture notes. The Kernel file can be found in your Laravel ```app/Http``` folder. Once you've registered your middleware, apply it to your ```/create``` route using the ```middleware()``` function. Restart your server, and verify that it kicks you back to the main landing page if you try to enter a chatroom name that falls outside the allowed validation (i.e. is too long, or too short).

There is no submission required for this lab. You will be graded based on the state of your server. This lab is worth a total of 9 marks - three marks each for correctly implementing the changes to your HTTP headers, creating a working Content Security Policy, and creating a working validation middleware for your chatroom names.

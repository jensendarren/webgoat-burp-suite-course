# WebGoat / Burp Suite Course

## Start WebGoat

`docker run -p 8080:8080 -it webgoat/webgoat-8.0 /home/webgoat/start.sh`

## Stop WebGoat

Importantly, the container should be stopped this way:

`docker container stop <CONTAINER-ID>`

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






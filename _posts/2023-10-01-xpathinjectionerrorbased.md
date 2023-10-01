---
title: "XPATH Injection - Exploiting Error-based SQL Injection"
date: 2023-10-01 18:00:00 +0900
categories: [mobile]
tags: [mobile, sqlinjection]
comments: false
---

Hey hackers!! Today, I'm explaining the exploitation of the error-based SQL injection via XPATH injection. This vulnerability was discovered during a private pen-test engagement.

---

### Understanding the UPDATEXML and EXTRACTVALUE Functions

* Let's first understand the **UPDATEXML** and **EXTRACTVALUE** XML functions, as they are helpful for exploiting the error-based SQL injection. 

> Here, we are taking reference of the **MySQL** XML functions.
{: .prompt-info }

1. **UPDATEXML()** - This function is used to update XML data within an XML type column. The syntax of the function is given below.

     > `UpdateXML(xml_target, xpath_expr, new_xml)`
 
    * Explaination of the syntax,
    > 1. `xml_target` - This represents the XML data/column which needs to be updated.
    > 2. `xpath_expr` - An XPath Expression which specifies the location of the XML element which needs to update witin `xml_target`.
    > 3. `new_xml` - This represents new XML element which will be replaced/updated.

2. **EXTRACTVALUE** - This function is used to extract a value from an XML document at a specified XPath expression.

    > `ExtractValue(xml_frag, xpath_expr)`

    * Explaination of the syntax,
    > 1. `xml_frag` - This represents the XML fragment from which you want to extract a value.
    > 2. `xpath_expr` - The use case is similar to what was described above in the **UPDATEXML** syntax.

* Now we have general idea of these functions, let's dive into the actual exploitation.


### Discovery

* The application was an Android-based payment application for merchants. Merchant users could log in to the application to send the requests for payment, check the sales of the month and access the dashboard. I initiated testing by starting the frida server and intercepting API requests in the burpsuite proxy.
* While checking the requests, I came across a **GET** API request with parameters that appear as follows. You can also view the response of this request in the screenshot.

`https://example.com/jounral/dashboard/sales?pageNumber=0&pageSize=20&fromDate=2021-01-01&toDate=2023-06-27&search`

![POC1xpath](/assets/img/POC1.png)
_HTTPRequest_


### Exploitation

1. Looking at the parameters `fromDate` and `toDate`, there was possibility of the SQL Injection. I simply added a single quote(`'`) as value for the `toDate` parameter and forwarded the request. The server responded with an error message `SQL ERROR`.

    ![POC2xpath](/assets/img/POC2.png)
    _HTTPRequestwithInjectedSingleQuote_

    * I attempted manual exploitation using **UNION-based, Boolean-based and Blind-based** SQL Injection payloads but none of them were successful. I was consistently receiving the `SQL ERROR` message. I also tried using the obfuscated payloads but that also didn't work and SQLMAP didn't provide any results.

2. Using the payload, `' updatexml(null,concat(0x0a,(select version())),null) '` provided the SQL error stack-trace with the entire query and also received **XPATH syntax error: Query result** in the response. It was observed that application was using `MariaDB` DBMS. I noticed that the **EXTRACTVALUE** function was filtered by firewall so we cannot use it.

    ![POC3xpath](/assets/img/POC4.png)
    _UPDATEXML func. to extract version and server_

    * If we observe the screenshot, the payload is injected in the middle of SQL query. Please note that if the XPath query is syntactically incorrect then only we will receive the error `XPATH syntax error:` . 
    * In the payload, `null` is the syntactically incorrect for the **UPDATEXML** function because it expects *first argument* as XML column and *third argument* as new XML content that we wants to update.

3. Instead of `version()` , I provided the `database()` to get the name of the current database.

    ![POC4xpath](/assets/img/POC5.png)
    _UPDATEXML func. to extract current databasename_


* It can be further exploited to exfiltrate tables of the current database by using the below payload.

```sql
updatexml(null,concat(0x3a,(select table_name from information_schema.tables where table_schema=database() limit 0,1)),null)--
```

### Impact

* While sometimes traditional SQL injection techniques may not be effective, attackers might utilize XML functions like **UPDATEXML** and **EXTRACTVALUE** to exploit SQL injection vulnerability and exfiltrate data from the database.


### References

* [MySQL XML Functions](https://dev.mysql.com/doc/refman/8.0/en/xml-functions.html)
* [XPATH Injection using UPDATEXML](https://raijee1337.blogspot.com/2015/07/xpath-injection-using-updatexml.html)

Thank you for taking time to read the blog. Happy Hacking!!!
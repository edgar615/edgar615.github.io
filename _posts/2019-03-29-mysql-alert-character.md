---
layout: post
title: Mysql修改数据库字符集
date: 2019-03-29
categories:
    - MySQL
comments: true
permalink: mysql-alert-character.html
---

修改数据库字符集:

<pre class="line-numbers "><code class="language-sql">
ALTER DATABASE db_name DEFAULT CHARACTER SET character_name [COLLATE ...]; 
</code></pre>
把表默认的字符集和所有字符列（CHAR,VARCHAR,TEXT）改为新的字符集：
<pre class="line-numbers "><code class="language-sql">
ALTER TABLE tbl_name CONVERT TO CHARACTER SET character_name [COLLATE ...]  
-- 示例
ALTER TABLE user CONVERT TO CHARACTER SET utf8 COLLATE utf8_general_ci;  
</code></pre>

只是修改表的默认字符集:
<pre class="line-numbers "><code class="language-sql">
ALTER TABLE tbl_name DEFAULT CHARACTER SET character_name [COLLATE...];  
-- 示例
ALTER TABLE user DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;   
</code></pre>

修改字段的字符集：

<pre class="line-numbers"><code class="language-sql">
ALTER TABLE tbl_name CHANGE c_name c_name CHARACTER SET character_name [COLLATE ...]; 
-- 示例
ALTER TABLE user CHANGE username username VARCHAR(100) CHARACTER SET utf8 COLLATE utf8_general_ci;  
</code></pre>

查看数据库编码

<pre class="line-numbers"><code class="language-sql">
SHOW CREATE DATABASE db_name;   
</code></pre>

查看表编码

<pre class="line-numbers"><code class="language-sql">
SHOW CREATE TABLE tbl_name;  
</code></pre>

查看字段编码

<pre class="line-numbers"><code class="language-sql">
SHOW FULL COLUMNS FROM tbl_name;  
</code></pre>

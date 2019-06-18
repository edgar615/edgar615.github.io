---
layout: post
title: MySQL插入大量测试数据
date: 2019-06-18
categories:
    - MySQL
comments: true
permalink: mysql-load-large-data.html
---

平时在测试SQL性能的时候要插入大量的模拟数据，通过JDBC插入通常耗时很久。搜索了一下找到了一个解决方案`LOAD DATA INFILE`

```
LOAD DATA [LOW_PRIORITY | CONCURRENT] [LOCAL] INFILE 'file_name'
    [REPLACE | IGNORE]
    INTO TABLE tbl_name
    [CHARACTER SET charset_name]
    [{FIELDS | COLUMNS}
        [TERMINATED BY 'string']
        [[OPTIONALLY] ENCLOSED BY 'char']
        [ESCAPED BY 'char']
    ]
    [LINES
        [STARTING BY 'string']
        [TERMINATED BY 'string']
    ]
    [IGNORE number LINES]
    [(col_name_or_user_var,...)]
    [SET col_name = expr,...]
```
先通过程序生成一个数据文件，每一行为一条记录，然后再使用上面提到的 `LOAD DATA INFILE` 来导入数据就可以了

```
  File file = new File("e:/test.txt");

  if (file.exists()) {
	file.delete();
  }
  file.createNewFile();

  FileOutputStream outStream = new FileOutputStream(file, false);

  FileWriter fw = new FileWriter(file.getAbsoluteFile());
  BufferedWriter bw = new BufferedWriter(fw);

  Random rand = new Random();


  int i = 1000000;
  while (i++ < 10000000) {
	StringBuilder builder = new StringBuilder();
	//订单号
	builder.append(i);
	builder.append("\t");
	//卖家ID
	builder.append(Math.abs(rand.nextInt(999)));
	builder.append("\t");
	//卖家名称
	builder.append(Math.abs(rand.nextInt(999)));
	builder.append("\t");
	//买家ID
	builder.append(Math.abs(rand.nextInt(999)));
	builder.append("\t");
	//买家名称
	builder.append(Math.abs(rand.nextInt(999)));
	builder.append("\t");

	//订单金额
	builder.append(Math.abs(rand.nextInt(99999999)));
	builder.append("\t");
	//状态
	builder.append(Math.abs(rand.nextInt(5)));
	builder.append("\t");
	//时间
	builder.append(Instant.now().getEpochSecond() + rand.nextInt(9999));
	builder.append("\n");
	outStream.write(builder.toString().getBytes());
  }
  outStream.close();
```

我通过Navicat远程导入100万数据，用时10分钟左右，命令学着太累，o(╥﹏╥)o



# LOAD DATA INFILE 详细介绍

网上摘抄的，没有排版

> LOAD DATA INFILE语句从一个文本文件中以很高的速度读入一个表中。如果指定LOCAL关键词，从客户主机读文件。如果LOCAL没指定，文件必须位于服务器上。(LOCAL在MySQL3.22.6或以后版本中可用。）  
>
> 为了安全原因，当读取位于服务器上的文本文件时，文件必须处于数据库目录或可被所有人读取。另外，为了对服务器上文件使用LOAD DATA INFILE，在服务器主机上你必须有file的权限。见6.5 由MySQL提供的权限。  
>
> 如果你指定关键词LOW_PRIORITY，LOAD DATA语句的执行被推迟到没有其他客户读取表后。  
>
> 使用LOCAL将比让服务器直接存取文件慢些，因为文件的内容必须从客户主机传送到服务器主机。在另一方面，你不需要file权限装载本地文件。  
>
> 你也可以使用mysqlimport实用程序装载数据文件；它由发送一个LOAD DATA INFILE命令到服务器来运作。 --local选项使得mysqlimport从客户主机上读取数据。如果客户和服务器支持压缩协议，你能指定--compress在较慢的网络上获得更好的性能。  
>
> 当在服务器主机上寻找文件时，服务器使用下列规则：  
>
> 如果给出一个绝对路径名，服务器使用该路径名。   
> 如果给出一个有一个或多个前置部件的相对路径名，服务器相对服务器的数据目录搜索文件。   
> 如果给出一个没有前置部件的一个文件名，服务器在当前数据库的数据库目录寻找文件。   
> 注意这些规则意味着一个像“./myfile.txt”给出的文件是从服务器的数据目录读取，而作为“myfile.txt”给出的一个文件是从当前数据库的数据库目录下读取。也要注意，对于下列哪些语句，对db1文件从数据库目录读取，而不是db2：  
>
> mysql> USE db1;  
> mysql> LOAD DATA INFILE "./data.txt" INTO TABLE db2.my_table;  
>
> REPLACE和IGNORE关键词控制对现有的唯一键记录的重复的处理。如果你指定REPLACE，新行将代替有相同的唯一键值的现有行。如果你指定IGNORE，跳过有唯一键的现有行的重复行的输入。如果你不指定任何一个选项，当找到重复键键时，出现一个错误，并且文本文件的余下部分被忽略时。  
>
> 如果你使用LOCAL关键词从一个本地文件装载数据，服务器没有办法在操作的当中停止文件的传输，因此缺省的行为好像IGNORE被指定一样。  
>
> LOAD DATA INFILE是SELECT ... INTO OUTFILE的逆操作，  
> SELECT句法。为了将一个数据库的数据写入一个文件，使用SELECT ... INTO OUTFILE，为了将文件读回数据库，使用LOAD DATA INFILE。两个命令的FIELDS和LINES子句的语法是相同的。两个子句是可选的，但是如果指定两个，FIELDS必须在LINES之前。  
>
> 如果你指定一个FIELDS子句，它的每一个子句(TERMINATED BY, [OPTIONALLY] ENCLOSED BY和ESCAPED BY)也是可选的，除了你必须至少指定他们之一。  
>
> 如果你不指定一个FIELDS子句，缺省值与如果你这样写的相同：  
>
> FIELDS TERMINATED BY '\t' ENCLOSED BY '' ESCAPED BY '\\'  
>
> 如果你不指定一个LINES子句，缺省值与如果你这样写的相同：  
>
> LINES TERMINATED BY '\n'   
> 换句话说，缺省值导致读取输入时，LOAD DATA INFILE表现如下：  
>
> 在换行符处寻找行边界   
> 在定位符处将行分进字段   
> 不要期望字段由任何引号字符封装   
> 将由“\”开头的定位符、换行符或“\”解释是字段值的部分字面字符   
> 相反，缺省值导致在写入输出时，SELECT ... INTO OUTFILE表现如下：  
>
> 在字段之间写定位符   
> 不用任何引号字符封装字段   
> 使用“\”转义出现在字段中的定位符、换行符或“\”字符   
> 在行尾处写换行符   
> 注意，为了写入FIELDS ESCAPED BY '\\'，对作为一条单个的反斜线被读取的值，你必须指定2条反斜线值。  
>
> IGNORE number LINES选项可被用来忽略在文件开始的一个列名字的头：  
>
> mysql> LOAD DATA INFILE "/tmp/file_name" into table test IGNORE 1 LINES;  
>
> 当你与LOAD DATA INFILE一起使用SELECT ... INTO OUTFILE将一个数据库的数据写进一个文件并且随后马上将文件读回数据库时，两个命令的字段和处理选项必须匹配，否则，LOAD DATA INFILE将不能正确解释文件的内容。假定你使用SELECT ... INTO OUTFILE将由逗号分隔的字段写入一个文件：  
>
> mysql> SELECT * FROM table1 INTO OUTFILE 'data.txt'  
> ​           FIELDS TERMINATED BY ','  
> ​           FROM ...  
>
> 为了将由逗号分隔的文件读回来，正确的语句将是：  
>
> mysql> LOAD DATA INFILE 'data.txt' INTO TABLE table2  
> ​           FIELDS TERMINATED BY ',';  
>
> 相反，如果你试图用下面显示的语句读取文件，它不会工作，因为它命令LOAD DATA INFILE在字段之间寻找定位符：  
>
> mysql> LOAD DATA INFILE 'data.txt' INTO TABLE table2  
> ​           FIELDS TERMINATED BY '\t';  
>
> 可能的结果是每个输入行将被解释为单个的字段。  
>
> LOAD DATA INFILE能被用来读取从外部来源获得的文件。例如，以dBASE格式的文件将有由逗号分隔并用双引号包围的字段。如果文件中的行由换行符终止，下面显示的命令说明你将用来装载文件的字段和行处理选项：  
>
> mysql> LOAD DATA INFILE 'data.txt' INTO TABLE tbl_name  
> ​           FIELDS TERMINATED BY ',' ENCLOSED BY '"'  
> ​           LINES TERMINATED BY '\n';  
>
> 任何字段或行处理选项可以指定一个空字符串('')。如果不是空，FIELDS [OPTIONALLY] ENCLOSED BY和FIELDS ESCAPED BY值必须是一个单个字符。FIELDS TERMINATED BY和LINES TERMINATED BY值可以是超过一个字符。例如，写入由回车换行符对（CR+LF）终止的行，或读取包含这样行的一个文件，指定一个LINES TERMINATED BY '\r\n'子句。  
>
> FIELDS [OPTIONALLY] ENCLOSED BY控制字段的包围字符。对于输出(SELECT ... INTO OUTFILE)，如果你省略OPTIONALLY，所有的字段由ENCLOSED BY字符包围。对于这样的输出的一个例子(使用一个逗号作为字段分隔符)显示在下面：  
>
> "1","a string","100.20"  
> "2","a string containing a , comma","102.20"  
> "3","a string containing a \" quote","102.20"  
> "4","a string containing a \", quote and comma","102.20"  
>
> 如果你指定OPTIONALLY，ENCLOSED BY字符仅被用于包围CHAR和VARCHAR字段：  
>
> 1,"a string",100.20  
> 2,"a string containing a , comma",102.20  
> 3,"a string containing a \" quote",102.20  
> 4,"a string containing a \", quote and comma",102.20  
>
> 注意，一个字段值中的ENCLOSED BY字符的出现通过用ESCAPED BY字符作为其前缀来转义。也要注意，如果你指定一个空ESCAPED BY值，可能产生不能被LOAD DATA INFILE正确读出的输出。例如，如果转义字符为空，上面显示的输出显示如下。注意到在第四行的第二个字段包含跟随引号的一个逗号，它(错误地)好象要终止字段：  
>
> 1,"a string",100.20  
> 2,"a string containing a , comma",102.20  
> 3,"a string containing a " quote",102.20  
> 4,"a string containing a ", quote and comma",102.20  
>
> 对于输入，ENCLOSED BY字符如果存在，它从字段值的尾部被剥去。（不管是否指定OPTIONALLY都是这样；OPTIONALLY对于输入解释不起作用)由ENCLOSED BY字符领先的ESCAPED BY字符出现被解释为当前字段值的一部分。另外，出现在字段中重复的ENCLOSED BY被解释为单个ENCLOSED BY字符，如果字段本身以该字符开始。例如，如果ENCLOSED BY '"'被指定，引号如下处理：  
>
> "The ""BIG"" boss" -> The "BIG" boss  
> The "BIG" boss      -> The "BIG" boss  
> The ""BIG"" boss    -> The ""BIG"" boss  
>
> FIELDS ESCAPED BY控制如何写入或读出特殊字符。如果FIELDS ESCAPED BY字符不是空的，它被用于前缀在输出上的下列字符：  
>
> FIELDS ESCAPED BY字符   
> FIELDS [OPTIONALLY] ENCLOSED BY字符   
> FIELDS TERMINATED BY和LINES TERMINATED BY值的第一个字符   
> ASCII 0（实际上将后续转义字符写成 ASCII'0'，而不是一个零值字节）   
> 如果FIELDS ESCAPED BY字符是空的，没有字符被转义。指定一个空转义字符可能不是一个好主意，特别是如果在你数据中的字段值包含刚才给出的表中的任何字符。  
>
> 对于输入，如果FIELDS ESCAPED BY字符不是空的，该字符的出现被剥去并且后续字符在字面上作为字段值的一个部分。例外是一个转义的“0”或“N”（即，\0或\N，如果转义字符是“\”)。这些序列被解释为ASCII 0（一个零值字节）和NULL。见下面关于NULL处理的规则。  
>
> 对于更多关于“\”- 转义句法的信息，在某些情况下，字段和行处理选项相互作用：  
>
> 如果LINES TERMINATED BY是一个空字符串并且FIELDS TERMINATED BY是非空的，行也用FIELDS TERMINATED BY终止。   
> 如果FIELDS TERMINATED BY和FIELDS ENCLOSED BY值都是空的('')，一个固定行(非限定的)格式被使用。用固定行格式，在字段之间不使用分隔符。相反，列值只用列的“显示”宽度被写入和读出。例如，如果列被声明为INT(7)，列的值使用7个字符的字段被写入。对于输入，列值通过读取7个字符获得。固定行格式也影响NULL值的处理；见下面。注意如果你正在使用一个多字节字符集，固定长度格式将不工作。   
> NULL值的处理有多种，取决于你使用的FIELDS和LINES选项：  
>
> 对于缺省FIELDS和LINES值，对输出，NULL被写成\N，对输入，\N被作为NULL读入(假定ESCAPED BY字符是“\”)。   
> 如果FIELDS ENCLOSED BY不是空的，包含以文字词的NULL作为它的值的字段作为一个NULL值被读入(这不同于包围在FIELDS ENCLOSED BY字符中的字NULL，它作为字符串'NULL'读入)。   
> 如果FIELDS ESCAPED BY是空的，NULL作为字NULL被写入。   
> 用固定行格式(它发生在FIELDS TERMINATED BY和FIELDS ENCLOSED BY都是空的时候)，NULL作为一个空字符串被写入。注意，在写入文件时，这导致NULL和空字符串在表中不能区分，因为他们都作为空字符串被写入。如果在读回文件时需要能区分这两者，你应该不使用固定行格式。   
> 一些不被LOAD DATA INFILE支持的情况：  
>
> 固定长度的行(FIELDS TERMINATED BY和FIELDS ENCLOSED BY都为空)和BLOB或TEXT列。   
> 如果你指定一个分隔符与另一个相同，或是另一个的前缀，LOAD DATA INFILE不能正确地解释输入。例如，下列FIELDS子句将导致问题：   
> FIELDS TERMINATED BY '"' ENCLOSED BY '"'  
>
> 如果FIELDS ESCAPED BY是空的，一个包含跟随FIELDS TERMINATED BY值之后的FIELDS ENCLOSED BY或LINES TERMINATED BY的字段值将使得LOAD DATA INFILE过早地终止读取一个字段或行。这是因为LOAD DATA INFILE不能正确地决定字段或行值在哪儿结束。   
> 下列例子装载所有persondata表的行：  
>
> mysql> LOAD DATA INFILE 'persondata.txt' INTO TABLE persondata;  
>
> 没有指定字段表，所以LOAD DATA INFILE期望输入行对每个表列包含一个字段。使用缺省FIELDS和LINES值。  
>
> 如果你希望仅仅装载一张表的某些列，指定一个字段表：  
>
> mysql> LOAD DATA INFILE 'persondata.txt'  
> ​           INTO TABLE persondata (col1,col2,...);  
>
> 如果在输入文件中的字段顺序不同于表中列的顺序，你也必须指定一个字段表。否则，MySQL不能知道如何匹配输入字段和表中的列。  
>
> 如果一个行有很少的字段，对于不存在输入字段的列被设置为缺省值。  
>
> 如果字段值缺省，空字段值有不同的解释：  
>
> 对于字符串类型，列被设置为空字符串。   
> 对于数字类型，列被设置为0。   
> 对于日期和时间类型，列被设置为该类型的适当“零”值。   
> 如果列有一个NULL，或(只对第一个TIMESTAMP列)在指定一个字段表时，如果TIMESTAMP列从字段表省掉，TIMESTAMP列只被设置为当前的日期和时间。  
>
> 如果输入行有太多的字段，多余的字段被忽略并且警告数字加1。  
>
> LOAD DATA INFILE认为所有的输入是字符串，因此你不能像你能用INSERT语句的ENUM或SET列的方式使用数字值。所有的ENUM和SET值必须作为字符串被指定！  
>
> 如果你正在使用C API，当LOAD DATA INFILE查询完成时，你可通过调用API函数mysql_info()得到有关查询的信息。信息字符串的格式显示在下面：  
>
> Records: 1 Deleted: 0 Skipped: 0 Warnings: 0  
> 当值通过INSERT语句插入时，在某些情况下出现警告，除了在输入行中有太少或太多的字段时，LOAD DATA INFILE也产生警告。警告没被存储在任何地方；警告数字仅能用于表明一切是否顺利。如果你得到警告并且想要确切知道你为什么得到他们，一个方法是使用SELECT ... INTO OUTFILE到另外一个文件并且把它与你的原版输入文件比较。  
---
# author:
title: OpenGauss环境配置与数据库操作实验
description: >-
  学校实验课强制要求用OpenGauss进行实验，特此记录下环境配置以及实验课内容
date: 2025-05-28 22:33:00 +0800
categories: [学科笔记, 数据库系统]
tags: [数据库系统, 环境配置]
# pin: true
# media_subpath: '/resources/'
# render_with_liquid: false
math: true
# mermaid: true
# image:
#   path: /resources/xxxxxx.png
#   lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
#   alt: Responsive rendering of Chirpy theme on multiple devices
---

## 一、关于OpenGauss
- [OpenGauss官网](https://opengauss.org/zh/)

## 二、Docker环境配置

### 2.1 容器创建与启动
- 参考[官方文档](https://docs.opengauss.org/zh/docs/7.0.0-RC1/docs/InstallationGuide/%E5%AE%B9%E5%99%A8%E9%95%9C%E5%83%8F%E5%AE%89%E8%A3%85.html)

```bash
docker run --name TestOpenGauss --privileged=true -d -e GS_PASSWORD=Your_Robust_Password -p 8888:5432 opengauss/opengauss-server:latest
```

- 创建好容器后，运行并进入该容器

```bash
docker start 容器ID或容器名
docker exec -it 容器ID或容器名 bash
```

![运行并进入OpenGauss容器.png](/resources/2025-05-28-OpenGauss环境配置与数据库操作实验/运行并进入OpenGauss容器.png)

### 2.2 切换至超级用户
- 上图中正处于`root`的用户环境，而`\dt`等指令是OpenGauss的`gsql`客户端命令（后者类似于MySQL的`mysql`或PostgreSQL的`psql`）而非Linux系统命令，故我们需要切换到`omm`超级用户，并通过`gsql`连接数据库

```bash
su omm
gsql -d postgres -p 5432
```

![OpenGauss的omm用户和gsql连接.png](/resources/2025-05-28-OpenGauss环境配置与数据库操作实验/OpenGauss的omm用户和gsql连接.png)

## 三、测试数据库操作

### 3.1 查询数据库
- 通过以下指令可查看当前数据库（DB）的名称

```postgresql
SELECT current_database();
```

- 通过以下两种指令之一可以查询所有数据库

```postgresql
\l
SELECT datname FROM pg_database;
```

![查看所有DB.png](/resources/2025-05-28-OpenGauss环境配置与数据库操作实验/查看所有DB.png)

- 可以通过以下指令切换到特定数据库，切换成功后命令提示符会变为`db_name=#`

```postgresql
\c db_name
```

- 然后可通过以下指令查询当前数据库信息

```postgresql
-- 查看所有已创建的表
\dt

-- 查看表结构
\d Club
\d Student
\d ClubParticipation

-- 查看表中的数据
SELECT * FROM Student;
```

![查询数据库中的表相关信息.png](/resources/2025-05-28-OpenGauss环境配置与数据库操作实验/查询数据库中的表相关信息.png)

### 3.2 创建数据库
- 上述示例中提到了一个`Club`数据库，通过后续笔记内容我们即可复现该数据库，我们首先创建该DB本身

```postgresql
CREATE DATABASE Club;
```

- 然后切换到该数据库

```postgresql
\c Club
```

### 3.3 创建表及其约束
- 分别创建名为`Club`（恰与DB同名）、`Student`、`ClubParticipation`的表结构

```postgresql
-- 创建Club表，包含社团ID、名称和活动地点字段
CREATE TABLE Club(
	-- 社团ID，可变长度字符串，最多4个字符
    ClubID VARCHAR(4),
    Name VARCHAR(20),
    ActivityLocation VARCHAR(40)
);

-- 创建Student表，包含学生ID、姓名、性别和出生日期字段
CREATE TABLE Student(
    StudentID VARCHAR(7),
    Name VARCHAR(6),
    Gender VARCHAR(1),
    -- 出生日期，DATE类型
    BirthDate DATE
);

-- 创建ClubParticipation表，记录学生参加社团的情况
CREATE TABLE ClubParticipation(
    ClubID VARCHAR(4),
    StudentID VARCHAR(7),
    JoinDate DATE
);
```

- 上述`Student`的名字长度限制太短，可以改长点（注意分号`;`代表一条语句的结束）

```postgresql
ALTER TABLE Student
ALTER COLUMN name TYPE VARCHAR(20);
```

- 为上述三表分别设置主键

```postgresql
-- Club表主键为ClubID
ALTER TABLE Club ADD PRIMARY KEY (ClubID);
-- Student表主键为StudentID
ALTER TABLE Student ADD PRIMARY KEY (StudentID);
-- ClubParticipation表使用复合主键(ClubID,StudentID)
ALTER TABLE ClubParticipation ADD PRIMARY KEY (ClubID,StudentID);
```

- 在表创建后也可为表添加新列，例如以下指令添加`ManagerID`列到`Club`表中

```postgresql
ALTER TABLE Club ADD COLUMN ManagerID VARCHAR(7);
```

- 为上述三表分别设置外键约束

```postgresql
-- ClubParticipation表的ClubID引用Club表的ClubID
ALTER TABLE ClubParticipation ADD FOREIGN KEY (ClubID) REFERENCES Club(ClubID);
-- ClubParticipation表的StudentID引用Student表的StudentID
ALTER TABLE ClubParticipation ADD FOREIGN KEY (StudentID) REFERENCES Student(StudentID);
-- Club表的ManagerID引用Student表的StudentID
ALTER TABLE Club ADD FOREIGN KEY (ManagerID) REFERENCES Student(StudentID);
```

- 为上述三表分别设置其他约束

```postgresql
-- Club表的ActivityLocation不允许为空
ALTER TABLE Club ALTER COLUMN ActivityLocation SET NOT NULL;
-- Club表的Name字段必须唯一
ALTER TABLE Club ADD UNIQUE(Name);
-- Student表的Gender字段只能是'M'或'F'
ALTER TABLE Student ADD CHECK (Gender IN ('M', 'F'));
```

- 设置级联更新和级联删除
	- 级联更新：当==主表==的主键值被修改时，==从表==中对应的外键值会自动更新为相同值
	- 级联删除：当==主表==中的记录被删除时，==从表==中所有引用该记录的相关记录自动删除

```postgresql
ALTER TABLE ClubParticipation ADD FOREIGN KEY (ClubID) REFERENCES Club(ClubID) ON UPDATE CASCADE ON DELETE CASCADE;

ALTER TABLE ClubParticipation ADD FOREIGN KEY (StudentID) REFERENCES Student(StudentID) ON UPDATE CASCADE ON DELETE CASCADE;
```

### 3.4 插入和修改数据
- 以`Student`表为例，可为其插入数据，注意数据本身应当符合前文设置的规范

```postgresql
-- 向Student表插入一条记录，值顺序分别对应表结构StudentID, Name, Gender, BirthDate，其中BirthDate为NULL表示未知
INSERT INTO Student VALUES ('2021239', 'Hua Li', 'F', NULL);

-- 一次性插入多条数据
INSERT INTO Student VALUES 
('2021238', 'Wang Ming', 'M', '2000-12-29'),
('2021239', 'Chen Fei', 'F', '2001-01-24');
```

- 使用以下命令可通过主键`StudentID`对已插入的数据进行更新，此处是修改姓名

```postgresql
UPDATE Student SET Name = 'Nan Li' WHERE StudentID = '2021239';
```

- 使用以下命令可通过主键`StudentID`删除学生数据记录

```postgresql
DELETE FROM Student WHERE StudentID = '2021239';
```

- 再对其他表进行数据插入操作

```postgresql
-- 插入数据到Club表
INSERT INTO Club VALUES 
('0005', 'Roller Skating Club', 'Skating Rink at Sports Center', '2021239');

-- 插入数据到ClubParticipation表，其中注释的数据是不合法的，执行会报错如下
-- ERROR:  insert or update on table "clubparticipation" violates foreign key constraint "clubparticipation_studentid_fkey"
-- DETAIL:  Key (studentid)=(2021232) is not present in table "student".
INSERT INTO ClubParticipation VALUES
-- ('0005', '2021232', '2021-12-29'),
-- ('0005', '2021231', '2021-11-11'),
('0005', '2021238', '2021-12-15');
```

### 3.5 批量导入数据
- 在命令行（而非先前的`gsql`）中，将本地电脑存储数据的CSV文件复制到容器内

```bash
# 从Windows复制到容器（假设容器名为TestOpenGauss）
docker cp D:\Desktop\LabData\Student.csv TestOpenGauss:/tmp/
docker cp D:\Desktop\LabData\Club.csv TestOpenGauss:/tmp/
docker cp D:\Desktop\LabData\ClubParticipation.csv TestOpenGauss:/tmp/
```

![通过命令行导入CSV数据文件.png](/resources/2025-05-28-OpenGauss环境配置与数据库操作实验/通过命令行导入CSV数据文件.png)

- 然后进入`gsql`，过程中我们发现CSV文件中的日期是`dd/mm/yyyy`格式的，而OpenGauss只兼容`yyyy-mm-dd`格式的日期，所以需要进行转换，然后重新导入

```bash
docker cp D:\Desktop\LabData\Student-F.csv TestOpenGauss:/tmp/
docker cp D:\Desktop\LabData\ClubParticipation-F.csv TestOpenGauss:/tmp/
```

- 按如下顺序将上述CSV文件中的数据导入DB中的对应表结构中

```postgresql
-- 导入Student表（注意日期格式）
COPY Student FROM '/tmp/Student-F.csv' DELIMITER ',' CSV HEADER;

-- 导入Club表（若遇到社团名过长，则重新设置更大的名称上限即可）
COPY Club FROM '/tmp/Club.csv' DELIMITER ',' CSV HEADER;

-- 导入ClubParticipation表（注意日期格式）
COPY ClubParticipation FROM '/tmp/ClubParticipation-F.csv' DELIMITER ',' CSV HEADER;
```

- 导入完成后可通过以下代码检查导入的数据条数

```postgresql
SELECT COUNT(*) FROM student;
SELECT COUNT(*) FROM club;
SELECT COUNT(*) FROM clubparticipation;
```

![成功导入CSV数据到DB的表中.png](/resources/2025-05-28-OpenGauss环境配置与数据库操作实验/成功导入CSV数据到DB的表中.png)

- 也可通过以下方式进行检查，例如查看前五条数据

```postgresql
SELECT * FROM student LIMIT 5;
```

![查看前五条数据Student.png](/resources/2025-05-28-OpenGauss环境配置与数据库操作实验/查看前五条数据Student.png)

- 以下是原始数据，注意其中的日期格式，后文有用

```csv
//Student.csv
StudentID,Name,Gender,BirthDate
2021230,Wang Qiangzhuang,F,01/07/1999
2021231,Li Jingming,M,25/07/2006
2021232,Zhang Weiyu,M,04/12/2003
2021233,Zhao Tingchuan,M,02/04/2005
2021234,Liu Yanghui,M,16/11/1999
2021235,Chen Lioushiyin,M,25/11/1999
2021236,Zhou Xin,M,06/04/2003
2021237,Sun Lei,M,04/01/1999
2021250,Ma Li,F,01/05/2006
2021251,Wei Chen,M,30/01/2001
2021240,Feng Cheng,F,13/05/2000
2021241,Ding Lei,M,02/12/2002
2021242,Qian Kun,F,20/10/2005
2021243,Jiang Xue,F,02/09/2001
2021244,Gao Fei,F,15/06/2000
2021245,Song Qian,F,08/03/2001
2021246,Tian Yu,F,17/08/2006
2021247,Hu Yang,M,24/05/1999
2021248,Peng Lei,F,03/07/2002
2021249,Jia Meng,F,

//Club.csv
ClubID,Name,ActivityLocation,ManagerID
1000,Basketball Club,Activity Center Hall A,2021247
1001,Soccer Club,Activity Center Hall B,2021236
1002,Volleyball Club,Activity Center Hall C,2021237
1003,Table Tennis Club,Activity Center Hall D,2021246
1004,Badminton Club,Activity Center Hall E,2021246
1005,Animal Protection Club,Activity Center Hall F,2021248
1006,Calligraphy Club,Activity Center Hall G,2021236
1007,Painting Club,Activity Center Hall H,2021243
1008,Music Club,Activity Center Hall I,2021242
1009,Dance Club,Activity Center Hall J,2021230
1010,Photography Club,Activity Center Hall K,2021230
1011,Computer Club,Activity Center Hall L,2021240
1012,Literature Club,Activity Center Hall M,2021234
1013,Go Club,Activity Center Hall N,2021232
1014,Chinese Chess Club,Activity Center Hall O,2021249
1015,Rubik’s Cube Club,Activity Center Hall P,2021245
1016,Astronomy Club,Activity Center Hall Q,2021242
1017,Handicraft Club,Activity Center Hall R,2021241
1018,Model Club,Activity Center Hall S,2021247
1019,Anime Club,Activity Center Hall T,2021243

//ClubParticipation.csv
ClubID,StudentID,JoinDate
1009,2021246,20/05/2021
1010,2021249,15/01/2021
1018,2021248,28/03/2021
1014,2021237,15/08/2022
1002,2021244,18/01/2021
1006,2021230,16/10/2021
1007,2021243,03/04/2021
1014,2021234,21/03/2021
1017,2021235,22/03/2021
1011,2021231,22/06/2022
1011,2021230,23/10/2021
1009,2021244,21/01/2021
1018,2021241,01/11/2021
1011,2021237,31/05/2021
1015,2021243,24/10/2021
1001,2021238,01/05/2022
1004,2021249,30/10/2022
1005,2021240,07/05/2022
1007,2021234,05/05/2021
1016,2021230,15/08/2021
```

### 3.6 定制化查询任务
- 查询学号、姓名和性别

```postgresql
SELECT studentid, name, gender FROM student;
/*
 studentid |       name       | gender
-----------+------------------+--------
 2021238   | Wang Ming        | M
 2021239   | Chen Fei         | F
 2021230   | Wang Qiangzhuang | F
 2021231   | Li Jingming      | M
 2021232   | Zhang Weiyu      | M
 2021233   | Zhao Tingchuan   | M
 2021234   | Liu Yanghui      | M
 2021235   | Chen Lioushiyin  | M
 2021236   | Zhou Xin         | M
 2021237   | Sun Lei          | M
 2021250   | Ma Li            | F
 2021251   | Wei Chen         | M
 2021240   | Feng Cheng       | F
 2021241   | Ding Lei         | M
 2021242   | Qian Kun         | F
 2021243   | Jiang Xue        | F
 2021244   | Gao Fei          | F
 2021245   | Song Qian        | F
 2021246   | Tian Yu          | F
 2021247   | Hu Yang          | M
 2021248   | Peng Lei         | F
 2021249   | Jia Meng         | F
(22 rows)
+/
```

- 查询2002年前出生的学生学号

```postgresql
SELECT studentid FROM student WHERE birthdate < '2002-01-01';
/*
 studentid
-----------
 2021238
 2021239
 2021230
 2021234
 2021235
 2021237
 2021251
 2021240
 2021243
 2021244
 2021245
 2021247
(12 rows)
*/
```

- 查询2002年前出生的女生学号

```postgresql
SELECT studentid FROM student WHERE gender = 'F' AND birthdate < '2002-01-01';
/*
 studentid
-----------
 2021239
 2021230
 2021240
 2021243
 2021244
 2021245
(6 rows)
*/
```

- 查询年龄18-21岁的学生学号和姓名

```postgresql
SELECT studentid, name FROM student WHERE EXTRACT(YEAR FROM AGE(birthdate)) BETWEEN 18 AND 21;
/*
 studentid |      name
-----------+----------------
 2021231   | Li Jingming
 2021232   | Zhang Weiyu
 2021233   | Zhao Tingchuan
 2021250   | Ma Li
 2021242   | Qian Kun
 2021246   | Tian Yu
(6 rows)
*/
```

- 查询姓"张"的学生

```postgresql
SELECT studentid, name FROM student
WHERE name LIKE '张%' OR name LIKE 'Zhang%';
/*
 studentid |    name
-----------+-------------
 2021232   | Zhang Weiyu
(1 row)
*/
```

- 查询出生日期为NULL的学生

```postgresql
SELECT studentid, name FROM student WHERE birthdate IS NULL;
/*
 studentid |   name
-----------+----------
 2021249   | Jia Meng
(1 row)
*/
```

- 按性别升序、学号降序显示前5名学生

```postgresql
SELECT * FROM student ORDER BY gender ASC, studentid DESC LIMIT 5;
/*
 studentid |   name    | gender |      birthdate
-----------+-----------+--------+---------------------
 2021250   | Ma Li     | F      | 2006-05-01 00:00:00
 2021249   | Jia Meng  | F      |
 2021248   | Peng Lei  | F      | 2002-07-03 00:00:00
 2021246   | Tian Yu   | F      | 2006-08-17 00:00:00
 2021245   | Song Qian | F      | 2001-03-08 00:00:00
(5 rows)
*/
```

- 统计每个学生参与的社团数量

```postgresql
SELECT s.studentid, s.name, COUNT(cp.clubid) AS club_count
FROM student s
LEFT JOIN clubparticipation cp ON s.studentid = cp.studentid
GROUP BY s.studentid, s.name;
/*
 studentid |       name       | club_count
-----------+------------------+------------
 2021243   | Jiang Xue        |          2
 2021246   | Tian Yu          |          1
 2021235   | Chen Lioushiyin  |          1
 2021234   | Liu Yanghui      |          2
 2021251   | Wei Chen         |          0
 2021250   | Ma Li            |          0
 2021231   | Li Jingming      |          1
 2021247   | Hu Yang          |          0
 2021239   | Chen Fei         |          0
 2021241   | Ding Lei         |          1
 2021237   | Sun Lei          |          2
 2021248   | Peng Lei         |          1
 2021249   | Jia Meng         |          2
 2021230   | Wang Qiangzhuang |          3
 2021232   | Zhang Weiyu      |          0
 2021240   | Feng Cheng       |          1
 2021236   | Zhou Xin         |          0
 2021242   | Qian Kun         |          0
 2021244   | Gao Fei          |          2
 2021233   | Zhao Tingchuan   |          0
 2021238   | Wang Ming        |          2
 2021245   | Song Qian        |          0
(22 rows)
*/
```

- 统计每个社团的学生人数

```postgresql
SELECT c.clubid, c.name, COUNT(cp.studentid) AS student_count
FROM club c
LEFT JOIN clubparticipation cp ON c.clubid = cp.clubid
GROUP BY c.clubid, c.name;
/*
 clubid |          name          | student_count
--------+------------------------+---------------
 1004   | Badminton Club         |             1
 1013   | Go Club                |             0
 1000   | Basketball Club        |             0
 1006   | Calligraphy Club       |             1
 1016   | Astronomy Club         |             1
 1011   | Computer Club          |             3
 1019   | Anime Club             |             0
 1007   | Painting Club          |             2
 1002   | Volleyball Club        |             1
 0005   | Roller Skating Club    |             1
 1005   | Animal Protection Club |             1
 1003   | Table Tennis Club      |             0
 1001   | Soccer Club            |             1
 1010   | Photography Club       |             1
 1018   | Model Club             |             2
 1009   | Dance Club             |             2
 1008   | Music Club             |             0
 1015   | Rubik’s Cube Club      |             1
 1017   | Handicraft Club        |             1
 1012   | Literature Club        |             0
 1014   | Chinese Chess Club     |             2
(21 rows)
*/
```

### 3.7 角色与权限管理

```postgresql
-- 创建学生角色 WangMing（密码Abc*1234）
CREATE ROLE "WangMing" WITH PASSWORD 'Abc*1234' LOGIN;
-- 授予 Student 表查询和更新权限
GRANT SELECT, UPDATE ON Student TO "WangMing";

-- 创建社团管理员角色ChenFei（密码123）
CREATE ROLE "ChenFei" WITH PASSWORD 'Abc*1234' LOGIN;
-- 授予权限
GRANT SELECT ON Student TO "ChenFei";
GRANT SELECT, UPDATE ON Club, ClubParticipation TO "ChenFei";
```

![创建不同权限的数据库角色.png](/resources/2025-05-28-OpenGauss环境配置与数据库操作实验/创建不同权限的数据库角色.png)

- 输入密码登录对应角色

```bash
# 新终端测试角色登录（需映射端口8888）
gsql -d club -p 8888 -h 127.0.0.1 -U WangMing -W Abc*1234
# 若上面的指令报错且你懒得修复，可在容器内直接测试连接（绕过端口映射）
gsql -d club -p 5432 -U WangMing -W Abc*1234
```

- 尝试操作表，验证角色权限

```postgresql
-- 应成功
SELECT * FROM Student;
-- 应失败（无DELETE权限）
DELETE FROM Student;
```

![测试DB用户权限.png](/resources/2025-05-28-OpenGauss环境配置与数据库操作实验/测试DB用户权限.png)
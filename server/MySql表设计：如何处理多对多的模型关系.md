## MySql 表设计：如何处理多对多的模型关系

在日常开发中，业务上经常会碰到多对多的模型关系。举个简单的例子，我们要做一个需求统计工具，现在有一张表 story，story 表每条数据代表一个需求。我们还有一张表 user，记录的是系统人员。现在，需求上有一个字段处理人，每个需求可以有多个处理人，每个人可以同时处理多个需求，那么需求和人员就是很明显的多对多模型了。

### 多对多表设计

通常针对这种对多对模型，我们的设计方案是用第三张表抽取出来，记录 story 表和 user 表的关联关系。例如我们在 story 和 user 外添加一张表 story_user,假设一个需求有三个处理人，我们就往 story_user 里插入三条数据。

例如我们的 story 表如下

```sql
CREATE TABLE `story` (
`story_id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
`story_name` VARCHAR ( 255 ) DEFAULT NULL,
PRIMARY KEY ( `story_id` ),
) ENGINE = INNODB AUTO_INCREMENT = 15 DEFAULT CHARSET = utf8mb4 COLLATE = utf8mb4_0900_ai_ci;
```

我们的 user 表如下

```sql
CREATE TABLE `user` (
  `user_name` varchar(255) DEFAULT NULL,
  `user_id` int unsigned NOT NULL AUTO_INCREMENT,
  PRIMARY KEY (`user_id`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

这时我们再建立一张表，记录需求与用户的关系

```sql
CREATE TABLE `story_connect` (
  `user_id` int unsigned DEFAULT NULL,
  `story_id` int unsigned DEFAULT NULL,
  KEY `user_id` (`user_id`),
  KEY `story_id` (`story_id`),
  CONSTRAINT `story_connect_ibfk_1` FOREIGN KEY (`user_id`) REFERENCES `user` (`user_id`),
  CONSTRAINT `story_connect_ibfk_2` FOREIGN KEY (`story_id`) REFERENCES `story` (`story_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

这种方法非常常见。除了需求、处理人外，向学生、老师，学生、班级，老师、班级都是典型的多对多关系。只有添加一张关联表就能记录这种模型关系。

以下是老师学生多对多模型的图解

![avatar](https://for-wp.obs.cn-south-1.myhuaweicloud.com:443/e3d1940e0eafdbeddd4c2cfeb028f196.webp?AccessKeyId=0FBSS4ODLZBACMCPJGBG&Expires=1710454024&Signature=X9dfsEsctNSDh43UjrfI5i0JEC4%3D)

### 多对多表模型之数据统一问题

现在我们回到刚刚 story 和 user 的例子。我们现在有三张表（story、user、story_user）来记录需求和用户的信息。那假设用户要新建一个需求，并且给需求指定一个处理人，这时我们首先要在 story 表插入一条数据来新建需求，再在 story_user 表插入一套数据，给刚新建的需求指定处理人。

从 sql 上看也是非常简单

```sql
INSERT INTO story
VALUES (value1,value2,value3,...);

INSERT INTO story_user
VALUES (value1,value2,value3,...);
```

正常来说，这个流程走完是没问题的。但假如我们在执行第二条 sql 的时候，系统出错了导致插入失败，会出现什么情况？这时就导致成功建立了需求，但没有指定处理人。而这时如果你告诉前端，需求已经新建成功，那用户会很自然地认为该需求已经绑好了处理人。如果你告诉用户新建失败，story 里就多了一条冗余数据。难道多个 sql 的操作里，有一条失败的话，就要把前面的更新操作全部撤销才能保证业务合理吗？

没错，这种情况本就应该就行回滚。但撤销与回滚依靠的不是继续跑 sql，而是我们接下来要介绍的**事务**

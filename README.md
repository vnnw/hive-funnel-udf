# Hive Funnel Analysis UDFs

[![Build Status](https://travis-ci.org/yahoo/hive-funnel-udf.svg?branch=master)](https://travis-ci.org/yahoo/hive-funnel-udf)
[![Coverage Status](https://coveralls.io/repos/github/yahoo/hive-funnel-udf/badge.svg?branch=master)](https://coveralls.io/github/yahoo/hive-funnel-udf?branch=master)
[![Apache License 2.0](https://img.shields.io/badge/license-Apache%202.0-blue.svg?style=flat)](LICENSE)

[Funnel analysis](https://en.wikipedia.org/wiki/Funnel_analysis) is a method for
tracking user conversion rates across actions. This enables detection of actions
causing high user fallout.

These Hive UDFs enables funnel analysis to be performed simply and easily on any
Hive table.

## Requirements

[Maven](https://maven.apache.org/index.html) is required to build the funnel UDFs.

需使用3.x版本，建议使用Orcale JDK进行编译。

## How to build

There is a provided `Makefile` with all the build targets.

pom.xml文件可能需要修改的几个地方：

```
        <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <hive.version>对应的hive版本</hive.version>
        <hadoop.version>1.2.1</hadoop.version>
        </properties>

        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.5</version>
            <configuration>
                <source>1.7 <-- 编译机器的JDK版本</source>
                <target>1.7 <-- Hadoop集群的JDK版本</target>
            </configuration>
        </plugin>
```

### Build JAR

```bash
make jar
```

This creates a `funnel.jar` in the `target/` directory.

### Register JAR with Hive

To use the funnel UDFs, you need to register it with Hive.

With temporary functions:

```sql
ADD JAR funnel.jar;
CREATE TEMPORARY FUNCTION funnel         AS 'com.yahoo.hive.udf.funnel.Funnel';
CREATE TEMPORARY FUNCTION funnel_merge   AS 'com.yahoo.hive.udf.funnel.Merge';
CREATE TEMPORARY FUNCTION funnel_percent AS 'com.yahoo.hive.udf.funnel.Percent';
```

With permenant functions you need to put the JAR on HDFS, and it will be registered with a database (you have to replace `DATABASE` and `PATH_TO_JAR` with your values):

```sql
CREATE FUNCTION DATABASE.funnel         AS 'com.yahoo.hive.udf.funnel.Funnel'  USING JAR 'hdfs:///PATH_TO_JAR/funnel.jar';
CREATE FUNCTION DATABASE.funnel_merge   AS 'com.yahoo.hive.udf.funnel.Merge'   USING JAR 'hdfs:///PATH_TO_JAR/funnel.jar';
CREATE FUNCTION DATABASE.funnel_percent AS 'com.yahoo.hive.udf.funnel.Percent' USING JAR 'hdfs:///PATH_TO_JAR/funnel.jar';
```

## How to use

There are three funnel UDFs provided: [`funnel`](#funnel),
[`funnel_merge`](#funnel_merge), [`funnel_percent`](#funnel_percent).

The [`funnel`](#funnel) UDF outputs an array of longs showing conversion rates
across the provided funnels.

The [`funnel_merge`](#funnel_merge) UDF merges multiple arrays of longs by
adding them together.

The [`funnel_percent`](#funnel_percent) UDF takes a raw count funnel result and
converts it to a percent change count.

There is no need to sort the data on timestamp, the UDF will take care of it. If
there is a collision in the timestamps, it then sorts on the action column.

### `funnel`
`funnel(action_column, timestamp_column, array(funnel_1), array(funnel_2), ...)`
  - Builds a funnel report applied to the `action_column`, sorted by the
    `timestamp_column`.
  - The funnels are arrays of the same type as the `action` column. This allows
    for multiple matches to move to the next funnel.
    - For example, funnel_1 could be `array('register_button',
      'facebook_invite_register')`. The funnel will match the first occurence
      of either of these actions and proceed to the next funnel.
  - You can have an arbitrary number of funnels.
  - The `timestamp_column` can be of any comparable type (Strings, Integers,
    Dates, etc).

### `funnel_merge`
`funnel_merge(funnel_column)`
  - Merges funnels. Use with funnel UDF.

### `funnel_percent`
`funnel_percent(funnel_column)`
  - Converts the result of a [`funnel_merge`](#funnel_merge) to percent change.
    Use with funnel and funnel_merge UDF.
  - For example, a result from [`funnel_merge`](#funnel_merge) could look like
    `[245, 110, 54, 13]`. This is result is in raw counts. If we pass this
    through [`funnel_percent`](#funnel_percent) then it would look like `[1.0,
    0.44, 0.49, 0.24]`.

### Examples

Assume a table `user_data`:

| action              | timestamp | user_id | gender |
|---------------------|-----------|---------|--------|
| signup_page         | 100       | 1       | f      |
| confirm_button      | 200       | 1       | f      |
| submit_button       | 300       | 1       | f      |
| signup_page         | 200       | 2       | m      |
| submit_button       | 400       | 2       | m      |
| signup_page         | 100       | 3       | f      |
| confirm_button      | 200       | 3       | f      |
| decline             | 200       | 3       | f      |
| ...                 | ...       | ...     | ...    |

根据上面的示例数据，创建hive表，加载示例数据：

```
CREATE TABLE test_table(action string, timestamp int, user_id int, gender string)
       ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t';
LOAD DATA LOCAL INPATH '/tmp/test.csv' OVERWRITE INTO TABLE test_table;
```

#### Simple funnel: `(signup OR email_signup) -> confirm -> submit`

```sql
SELECT funnel_merge(funnel)
FROM (SELECT funnel(action, timestamp, array('signup_page', 'email_signup'),
                                       array('confirm_button'),
                                       array('submit_button')) AS funnel
      FROM test_table
      GROUP BY user_id) t1;
```

Result: `[3, 2, 1]`

#### Simple funnel with percent: `signup -> confirm -> submit`

```sql
SELECT funnel_percent(funnel_merge(funnel))
FROM (SELECT funnel(action, timestamp, array('signup_page'),
                                       array('confirm_button'),
                                       array('submit_button')) AS funnel
      FROM test_table
      GROUP BY user_id) t1;
```

Result: `[1.0, 0.66, 0.5]`

#### Funnel with multiple groups: `signup -> confirm -> submit` by gender

```sql
SELECT gender, funnel_merge(funnel)
FROM (SELECT gender,
             funnel(action, timestamp, array('signup_page'),
                                       array('confirm_button'),
                                       array('submit_button')) AS funnel
      FROM test_table
      GROUP BY user_id, gender) t1
GROUP BY gender;
```
注：Yahoo Github上的HQL示例，语法有误，应该还要在外层添加一个GROUP BY gender;


Result: `m: [1, 0, 0], f: [2, 2, 1]`

#### Multiple parallel funnels: `signup -> confirm -> submit` and `signup -> decline`

```sql
SELECT funnel_merge(funnel1), funnel_merge(funnel2)
FROM (SELECT funnel(action, timestamp, array('signup_page'),
                                       array('confirm_button'),
                                       array('submit_button')) AS funnel1
             funnel(action, timestamp, array('signup_page'),
                                       array('decline')) AS funnel2
      FROM test_table
      GROUP BY user_id) t1;
```

Result: `[3, 2, 1] [3, 1]`

## License

[Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0)

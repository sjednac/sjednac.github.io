---
layout: post
title: "Adapting H2 for in-memory integration tests with SQL Server compatibility"
excerpt: "Improving performance, reliability and determinism of SQL Server integration tests by replacing the database engine with a custom H2 build."
tags: [h2, sql server, integration tests, database, testing, legacy]
modified: 2018-09-30
comments: true
---

**UPDATE:** All features described in this article are currently [included](https://github.com/h2database/h2database/issues/1338) in the official release of the **H2** database.

----

Slow and non-deterministic tests can become a real issue, when dealing with large, monolithic applications and pre-existing code, that depends on a centralized data model (i.e. a shared database with some "test" data). One common improvement is to use an in-memory database instead, and run each test suite in it's own, isolated environment with no external dependencies.

There are several advantages of taking this approach:

  * Tests become more expressive and are easier to debug, since all initial state needs an explicit declaration in the test code.
  * Test suite isolation makes parallel execution easier and more predictable. Similarly, explicit transaction management may not be mandatory in the test context anymore (e.g. some people like to `@Rollback` their integration tests, when using Spring).
  * Performance and reliability are improved, since no predefined database connection is required.
  * Testing environment is much easier to setup, since it can be spawned directly by Maven, Gradle or some other build tool.
  * It may improve portability of your development environment.

The main drawback is the compatibility factor of course, since some issues may be visible in the target environment only. Whether this an acceptable risk or not, depends entirely on your application, your deployment process and your general approach to integration testing of course.

Having this in mind, most in-memory databases work pretty well with various **JPA** implementations, such as **Hibernate**. It only becomes a challenge when you start working with native queries and/or legacy code, that implement some "custom" persistence solutions.

To solve these problems in one of my projects, I've extended the [H2](http://www.h2database.com) database with some **SQL Server** syntax support. You can find the source code [here](https://github.com/sbilinski/h2database/tree/sqlserver-compat) and a brief summary of the features below.

Usage should be rather straightforward - just include a `MODE=MSSQLServer` in the JDBC connection string, when instantiating a `DataSource`:

{% highlight java %}
HikariConfig cfg = new HikariConfig();
cfg.setDriverClassName("org.h2.Driver");
cfg.setJdbcUrl("jdbc:h2:mem:;MODE=MSSQLServer");
DataSource ds = new HikariDataSource(cfg);
{% endhighlight %}

## Supported features

### Using IDENTITY as an alias for AUTO_INCREMENT

A column definition in **SQL Server** might include an `IDENTITY` keyword for handling automatic id generation by the database. **H2** supports this syntax as well, but - unfortunately - it can't be combined with a `PRIMARY KEY` prefix in a column definition. For this reason, I've added an `AUTO_INCREMENT` alias for `IDENTITY`, which enables us to run the following type of statements in our code:

{% highlight sql %}
create table parent(id int primary key identity)
{% endhighlight %}

... or with a `NOT NULL` modifier:

{% highlight sql %}
create table parent(id int primary key not null identity)
{% endhighlight %}

### Discarding table hints

[Table hints](https://msdn.microsoft.com/en-us/library/ms187373.aspx) are special SQL expressions, which are meant to override the default behavior of the query optimizer. We don't need to support those in our tests, therefore we're going to ignore them entirely.

The following types of hints are currently supported by the implementation (ignored by the parser):

{% highlight sql %}
select * from parent with(nolock)
select * from parent with(nolock, index = id)
select * from parent with(nolock, index(id, name))
select * from parent p with(nolock) join child ch with(nolock) on ch.parent_id = p.id
{% endhighlight %}

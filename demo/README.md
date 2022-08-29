# Integrating Spring Boot Application with a Local Docker SQL Instance

## Learning Goals

- How to use PostgreSQL with Spring Data
- Implementing simple data model benchmarks

## Instructions

In this lab, we will be implementing a simple Spring Boot JPA model that uses our new local PostgreSQL instance.
Once that is up and running, we will be examining some implications from trivial data models examples using poor and best practices.

## How to use Postgres with Spring Data

At this point, you should be comfortable configuring Spring Boot applications to interact with a local in memory database.
The good news is that implementing the needed changes can be quite trivial for a simple setup; we'll be looking later at more
complex environment setups. The bad news is that you will be moving to dedicated database instances, which brings its own set
of care and maintenance issues. Depending on the expected ownership of applications, this may or may not fall onto
a dedicated Database Administration team. From the application side though, this tends to be quite the straightforward operation.

Let us start with a simple baseline Spring Boot [application](https://spring.io/guides/gs/rest-service/). Go ahead and clone
the repository in full, as we will be starting from that completed project, and building Database interactions on top.

``` text
git clone https://github.com/spring-guides/gs-rest-service.git
cd gs-rest-service
git checkout 5cbc686 # To rollback to Spring Boot 2.6.x
```

Open up the Maven project from `gs-rest-service/complete` in your IntelliJ environment, and launch the application. You should
now be able to open up a browser, or use curl to hit the running endpoint. We will be using curl in this demonstration.

``` text
curl http://localhost:8080/greeting
```
``` shell
{"id":1,"content":"Hello, World!"}%
```
``` text
curl http://localhost:8080/greeting
```
``` shell
{"id":2,"content":"Hello, World!"}%
```

You should see that this is a simple Hello World application with a counter. If you stop and rerun the application, you will
see that this counter state does not persist. Let's go ahead and implement this simple counter as a Postgres backed
Database element.

Go ahead and start by adding the needed dependencies into the `pom.xml` file

``` text
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>
  <dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
  </dependency>
```
 
We are going to start with a minimal viable counter. Let us create a new class under `src/main/java/com/example/restservice/Counter.java`,
and its database interface under `src/main/java/com/example/restservice/CounterRepository.java`

``` java
package com.example.restservice;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;

@Entity
public class Counter {
    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long count;

    protected Counter() {}

    public Counter(Long count) {
        this.count = count;
    }

    public Long incrementAndGet() {
        this.count += 1;
        return count;
    }
}
```

``` java
package com.example.restservice;

import org.springframework.data.repository.CrudRepository;

public interface CounterRepository extends CrudRepository<Counter, Long> {}
```

And now make a few modifications to `src/main/java/com/example/restservice/GreetingController.java` to implement the new persistent endpoint.

``` java
...
import org.springframework.beans.factory.annotation.Autowired;
...

    @Autowired
    private CounterRepository counterRepository;
    
    @GetMapping("/persistent_greeting")
    public Greeting persistentGreeting(@RequestParam(value = "name", defaultValue = "World") String name) {
        Counter persistentCounter = counterRepository.findById(1L).orElseGet(() -> new Counter(0L));
        Long persistentCounterValue = persistentCounter.incrementAndGet();
        counterRepository.save(persistentCounter);
        return new Greeting(persistentCounterValue, String.format(template, name));
    }
```

This should look familiar to you, as we are still using straightforward Spring Data JPA here. In fact, the only difference
for using PostgreSQL at this level will be with configuring the JPA datasource properties.
Let us take a look at those now. We'll have to create this new file in `src/main/resources/application.properties`

``` text
spring.datasource.url=jdbc:postgresql://localhost:5432/db_test
spring.datasource.username=postgres
spring.datasource.password=mysecretpassword

spring.jpa.properties.hibernate.dialect = org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.database=POSTGRESQL
spring.jpa.properties.hibernate.jdbc.lob.non_contextual_creation=true
```

When used for complex production applications though, there is additional tuning that may need to be done with the JDBC connection.
Documentation to these options are available at [https://jdbc.postgresql.org/documentation/head/connect.html](https://jdbc.postgresql.org/documentation/head/connect.html)


You should be able to rerun the application now, and see that the new endpoint persists if you restart the application

``` text
curl "http://localhost:8080/greeting"
```
``` shell
{"id":1,"content":"Hello, World!"}%
```
``` text
curl "http://localhost:8080/greeting"
```
``` shell
{"id":2,"content":"Hello, World!"}%
```
``` text
curl "http://localhost:8080/persistent_greeting"
```
``` shell
{"id":1,"content":"Hello, World!"}%
```
``` text
curl "http://localhost:8080/persistent_greeting"
```
``` shell
{"id":2,"content":"Hello, World!"}%
```
``` text
# Restart application
...
curl "http://localhost:8080/greeting"
```
``` shell
{"id":1,"content":"Hello, World!"}%
```
``` text
curl "http://localhost:8080/persistent_greeting"
```
``` shell
{"id":3,"content":"Hello, World!"}%
```

## Implementing Data Model benchmarks

Now that we have a working Database client, let's go ahead and expand a bit by putting together some simple data models and queries.
We'll start with four different models solving the same problem. Given a Haystack table with a large amount of hay objects and one needle object, select the needle.
This isn't too interesting of an example, but it will give us some interesting metrics to look at.

We are implementing:

1. Model 1: A Key-Value approach which could simulate a well formulated data model
2. Model 2: A naive method of foreign key search which could simulate instances of table joins spanning a wide array of data
3. Model 3: The first method, but with implementing indexes for faster searches
4. Model 4: The second method, but with implementing indexes for faster searches


Model 1

Table: haystack

| ID | UUID | VALUE |
|----|------|-------|
| 1  | aaa  | hay   |
| 2  | bbb  | hay   |
| 3  | ccc  | hay   |
| ...| ...  | ...   |
| n  | nnn  | needle|


Model 2

Table: haystackuuid

| ID | UUID |
|----|------|
| 1  | aaa  |
| 2  | bbb  |
| 3  | ccc  |
| ...| ...  |
| n  | nnn  |

Table: haystack

| ID | UUID | VALUE |
|----|------|-------|
| 1  | aaa  | hay   |
| 2  | bbb  | hay   |
| 3  | ccc  | hay   |
| ...| ...  | ...   |
| n  | nnn  | needle|


Model 3

Table: haystack

| ID | UUID | VALUE |
|----|------|-------|
| 1  | aaa  | hay   |
| 2  | bbb  | hay   |
| 3  | ccc  | hay   |
| ...| ...  | ...   |
| n  | nnn  | needle|

Index: haystack_value_idx


Model 4

Table: haystackuuid

| ID | UUID |
|----|------|
| 1  | aaa  |
| 2  | bbb  |
| 3  | ccc  |
| ...| ...  |
| n  | nnn  |

Index: haystackuuid_uuid_idx

Table: haystack

| ID | UUID | VALUE |
|----|------|-------|
| 1  | aaa  | hay   |
| 2  | bbb  | hay   |
| 3  | ccc  | hay   |
| ...| ...  | ...   |
| n  | nnn  | needle|

Index: haystack_value_idx



There is an incomplete client for this under `src` in this repo, and a complete working client in the `solution` branch.
Go ahead and implement the missing components in the provided demo application. Incomplete sections are denoted with `TODO`.
Once this is complete, you should be able to see the benchmarking output, and have a better understanding of just how critical selecting an appropriate data model can be.
You can upload a screenshot of the benchmark output as a submission.

As this example below shows, there can be an order of magnitude speedup by selecting the correct models. And this only grows as the table sizes increase.

``` text
curl "http://localhost:8080/benchmark"
```
``` shell
Needle has value 5f6a9814-ff78-444a-89d0-8520aa7d484e at id 1000
Seq Scan on haystack  (cost=0.00..11.75 rows=1 width=540) (actual time=0.126..0.126 rows=1 loops=1)
  Filter: ((value)::text = 'needle'::text)
  Rows Removed by Filter: 999
Planning Time: 0.034 ms
Execution Time: 0.135 ms
---------------------------------------
Needle has value 5f6a9814-ff78-444a-89d0-8520aa7d484e at id 1000
Hash Join  (cost=11.76..43.43 rows=8 width=564) (actual time=0.189..0.191 rows=1 loops=1)
  Hash Cond: (haystackuuid.uuid = haystack.uuid)
  ->  Seq Scan on haystackuuid  (cost=0.00..25.70 rows=1570 width=24) (actual time=0.004..0.060 rows=1000 loops=1)
  ->  Hash  (cost=11.75..11.75 rows=1 width=540) (actual time=0.063..0.065 rows=1 loops=1)
        Buckets: 1024  Batches: 1  Memory Usage: 9kB
        ->  Seq Scan on haystack  (cost=0.00..11.75 rows=1 width=540) (actual time=0.062..0.064 rows=1 loops=1)
              Filter: ((value)::text = 'needle'::text)
              Rows Removed by Filter: 999
Planning Time: 0.037 ms
Execution Time: 0.201 ms
---------------------------------------
Needle has value 5f6a9814-ff78-444a-89d0-8520aa7d484e at id 1000
Bitmap Heap Scan on haystack  (cost=4.04..11.62 rows=5 width=540) (actual time=0.004..0.005 rows=1 loops=1)
  Recheck Cond: ((value)::text = 'needle'::text)
  Heap Blocks: exact=1
  ->  Bitmap Index Scan on haystack_value_idx  (cost=0.00..4.04 rows=5 width=0) (actual time=0.002..0.002 rows=1 loops=1)
        Index Cond: ((value)::text = 'needle'::text)
Planning Time: 0.020 ms
Execution Time: 0.012 ms
---------------------------------------
Needle has value 5f6a9814-ff78-444a-89d0-8520aa7d484e at id 1000
Hash Join  (cost=11.68..32.68 rows=25 width=564) (actual time=0.131..0.132 rows=1 loops=1)
  Hash Cond: (haystackuuid.uuid = haystack.uuid)
  ->  Seq Scan on haystackuuid  (cost=0.00..17.00 rows=1000 width=24) (actual time=0.006..0.056 rows=1000 loops=1)
  ->  Hash  (cost=11.62..11.62 rows=5 width=540) (actual time=0.010..0.011 rows=1 loops=1)
        Buckets: 1024  Batches: 1  Memory Usage: 9kB
        ->  Bitmap Heap Scan on haystack  (cost=4.04..11.62 rows=5 width=540) (actual time=0.009..0.009 rows=1 loops=1)
              Recheck Cond: ((value)::text = 'needle'::text)
              Heap Blocks: exact=1
              ->  Bitmap Index Scan on haystack_value_idx  (cost=0.00..4.04 rows=5 width=0) (actual time=0.004..0.004 rows=1 loops=1)
                    Index Cond: ((value)::text = 'needle'::text)
Planning Time: 0.047 ms
Execution Time: 0.144 ms%
```

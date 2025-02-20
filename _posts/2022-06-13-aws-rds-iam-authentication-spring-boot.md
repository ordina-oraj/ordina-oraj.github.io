---
layout: post
authors: [yolan_vloeberghs]
title: 'RDS IAM Authentication with Spring Boot - secure password-less database authentication on AWS'
image: /img/2022-06-13-aws-rds-iam-authentication-spring-boot.jpeg
tags: [Spring Boot, AWS, Spring, Security]
category: Cloud
comments: true
---

- [Introduction](#introduction)
- [Database role grant](#database-role-grant)
- [AWS](#aws)
- [Spring Boot](#spring-boot)
- [Conclusion](#conclusion)

## Introduction
Database authentication in Spring Boot has traditionally always been very simple to implement and configure to your needs.
Whether it is with a custom data source, or just using the `spring.datasource` properties, it was always pretty straightforward to get a working database connection.
Handling the sensitive data such as database passwords in different environments can be tricky, but nowadays, there are enough solutions to tackle this problem in a secure and discrete way.

Fortunately, if you are deploying your application on the AWS cloud, you don't have to struggle passwords anymore or need a fancy tool, because you simply don't need a password.
You are now able to establish a database connection by authenticating through IAM. 
Note that this feature only works for MariaDB, MySQL and PostgreSQL. 

The feature works with "authentication tokens", which is a string of characters that is unique and generated by Amazon RDS.
One authentication token has a lifespan of 15 minutes and needs to be re-generated before or after it expires to keep the connection alive.

This very useful mechanic can deeply strengthen the security of your application by erasing the need of a password and therefore, destroying another opportunity to get your sensitive data in the wrong hands.

Note that you need to explicitly [enable IAM authentication](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.Enabling.html){:target="_blank" rel="noopener noreferrer"} before you can use this feature.

## Database role grant
Enabling IAM authentication alone is not enough.
When you create a new database user for your application, you also need to grant the rights to the database user to fetch an authentication token to use in the authentication process.

For Postgres, this can be done with the following SQL query:
```sql
CREATE USER <databaseuser> WITH LOGIN;
GRANT rds_iam TO databaseuser;
```

For MySQL:
```sql
CREATE USER <databaseuser> IDENTIFIED WITH AWSAuthenticationPlugin AS 'RDS';
```

## AWS
The application can only fetch a new authentication token if it has the right to do so.
Since we are running our application on an AWS environment, it only makes sense to make use of IAM roles.

```json
{
   "Version": "2012-10-17",
   "Statement": [
      {
         "Effect": "Allow",
         "Action": [
             "rds-db:connect"
         ],
         "Resource": [
             "arn:aws:rds-db:region:account-id:dbuser:DbiResourceId/db-user-name"
         ]
      }
   ]
}
```

If you are running your application on an EC2 instance, make sure that your EC2 instance has an IAM role attached to it which explicitly gives access to the `rds-db:connect` action.
On EKS (Kubernetes for AWS), you can use [IAM roles for service accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html){:target="_blank" rel="noopener noreferrer"}.


## Spring Boot
You will only need the [AWS SDK for RDS](https://mvnrepository.com/artifact/com.amazonaws/aws-java-sdk-rds){:target="_blank" rel="noopener noreferrer"} to fetch an authentication token, which you can add as a dependency in Maven by adding the following code block:

```xml
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>rds</artifactId>
    <version>2.20.5</version>
</dependency>
```

To achieve the token fetching in Spring Boot, I created a custom `HikariDataSource` which overrides the `getPassword()` method and used `RdsIamAuthTokenGenerator` and `GetIamAuthTokenRequest` to create a request for an authentication token from RDS.

```java
public class RDSIAMDataSource extends HikariDataSource {
    @Override
    public String getPassword() {
        // Alternatively, you can create a Bean declaration of RdsClient. For demo purposes, I have decided to instantiate it here.
        RdsClient rdsClient = RdsClient.builder()
                .region(new DefaultAwsRegionProviderChain().getRegion())
                .build();
        
        RdsUtilities rdsUtilities = rdsClient.utilities();

        URI jdbcUri = parseJdbcURL(getJdbcUrl());

        GenerateAuthenticationTokenRequest request = GenerateAuthenticationTokenRequest.builder()
                .username(getUsername())
                .hostname(jdbcUri.getHost())
                .port(jdbcUri.getPort())
                .build();

        return rdsUtilities.generateAuthenticationToken(request);
    }

    private URI parseJdbcURL(String jdbcUrl) {
        String uri = jdbcUrl.substring(5);
        return URI.create(uri);
    }
}
```

{:refdef: style="text-align: center; font-style: italic;"}
Note that for MariaDB, you do not need your own custom implementation as [they have an implementation of their own](https://mariadb.com/kb/en/mariadb-connector-j-250-release-notes/#aws-iam){:target="_blank" rel="noopener noreferrer"}. 
{: refdef}

To make sure that our connection does not drop due to authentication failures, we are going to set the maximum lifespan for the connection (remember that authentication tokens are only valid for 15 minutes).

Don't forget to set your database properties (host, port and username). 
**Important note:** IAM authentication to the database requires an SSL connection.

```properties
# Sets the max lifetime of each connection to 840000 ms (14 minutes). After 14 minutes, the application does a new request to RDS for a fresh authentication token
spring.datasource.hikari.max-lifetime=840000
# JDBC url with the properties 'ssl' and 'sslmode', this properties are bound to PostgreSQL instances
spring.datasource.url=jdbc:postgresql://<db-instance-name>.<random>.<region>.rds.amazonaws.com:<port>/<db-name>?ssl=true&sslmode=require
# Database user which has the 'rds_iam' role
spring.datasource.username=<db user>
```
With this implementation, the `getPassword()` method will be called every 14 minutes.

## Conclusion
Security is an important factor in many companies and environments, and this is definitely a recommended way of securely accessing your database from inside your application as it is a big improvement over the traditional username/password authentication method.
You don't have a password, which means you don't have to share it, so you have zero risk of exposing your password to unwanted parties. Not having a password also eliminates the need to manage passwords and to rotate passwords every now and then.
With the authentication token lifespan of 15 minutes, each generated token is secure and expires rapidly in case the token gets leaked.

While it is slightly more of a hassle to set it up, it is definitely worth your while to implement RDS IAM authentication, especially if you are already running your application and infrastructure on AWS.

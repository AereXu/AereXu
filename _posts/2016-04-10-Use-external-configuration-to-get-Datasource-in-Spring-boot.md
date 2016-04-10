---
author: "AereXu"
layout: post
category: Java
title: Use external configuration to get Datasource in Spring-boot
tagline: by Snail
tags: [SpringData,SpringBoot]
modified: 2016-04-10
---

Spring data gives us strong support JPA.
Spring boot simplified the configuration of Spring Data. The [netgloo's example](http://blog.netgloo.com/2014/10/27/using-mysql-in-spring-boot-via-spring-data-jpa-and-hibernate/) shows how to access data with only a profile __application.properties__.  
However, some people needs the datasource to be configurable throw external profile but not project inside profile.
Ok, we can define a Datasource ourselves to replace the auto generated one by spring boot;

<!--more-->
# First
Make a default Spring data configuration in profile `src/main/resources/application.properties` 
```
    spring.datasource.url = jdbc:postgresql://localhost:5432/bmsc
    spring.datasource.username = aere
    spring.datasource.password = aerexu
    
    spring.datasource.testWhileIdle = true
    spring.datasource.validationQuery = SELECT 1
    
    spring.jpa.properties.datasource.driver-class-name=org.postgresql.Driver
    spring.jpa.properties.hibernate.dialect = org.hibernate.dialect.PostgreSQL9Dialect
    
    # Show or not log for each sql query
    spring.jpa.show-sql = true
    
    # Hibernate ddl auto (create, create-drop, update)
    spring.jpa.hibernate.ddl-auto = update
    
    # Naming strategy
    spring.jpa.hibernate.naming-strategy = org.hibernate.cfg.ImprovedNamingStrategy
```

# Second
Make a data source specific configuration profile  `C:\Users\test\Workplace\config\SpringAll\datasource.properties`
```
spring.datasource.url = jdbc:postgresql://192.168.99.100:5432/bmsc
spring.datasource.username = aere
spring.datasource.password = aerexu
```

# Last
Create an configuration class __PersistenceJPAConfig__
<!--lang: java-->
```java
    package com.aere.spring.all.config.jpa;
    
    // Imports ...
    
    @Configuration
    @ComponentScan
    @PropertySource(value = {"classpath:/application.properties",
            "file:/C:\\Users\\test\\Workplace\\config\\SpringAll\\datasource.properties"},
            ignoreResourceNotFound = true)
    public class PersistenceJPAConfig {
    
        @Bean
        @ConfigurationProperties(prefix="spring.datasource")
        public DataSource dataSource() {
            return new DriverManagerDataSource();
        }
    }
```
Here we get two profiles, but only one will be counted. If both two exist, the last one counts. If only one exists, of course the exist one count.
With such method, we can use the project one for develop and the external one for release. 
  
That's all, very simple but need some little tricks.

# Plus:  
If you want the password in profile showed as encrypted, you can __extend__ the __DriverManagerDataSource__.
```java
public class EncryptedDriverManagerDataSource extends DriverManagerDataSource{
    ...
    public EncryptedDriverManagerDataSource(String url, String username, String password) {
		setUrl(url);
		setUsername(username);
        String decryptedPass = someDecryptMethod(password);
		setPassword(decryptedPass);
	}
	...
}
```
Then change the __DriverManagerDataSource__ to __EncryptedDriverManagerDataSource__ in __PersistenceJPAConfig__





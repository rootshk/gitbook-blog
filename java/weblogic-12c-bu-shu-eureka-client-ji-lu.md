# \[weblogic 12c]部署eureka client记录

```
Spring Boot: 2.1.9.RELEASE 
Spring Cloud: Greenwich.SR3 
Weblogic: 12c(12.2.1.3.0)
```

#### 重点

**MAVEN POM文件**

```
<packaging>war</packaging>
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-tomcat</artifactId>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        <exclusions>
            <exclusion>
                <groupId>com.sun.jersey</groupId>
                <artifactId>jersey-client</artifactId>
            </exclusion>
            <exclusion>
                <groupId>com.sun.jersey</groupId>
                <artifactId>jersey-core</artifactId>
            </exclusion>
            <exclusion>
                <groupId>com.sun.jersey.contribs</groupId>
                <artifactId>jersey-apache-client4</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
    <dependency>
        <groupId>javax.ws.rs</groupId>
        <artifactId>jsr311-api</artifactId>
        <version>1.1.1</version>
        <scope>provided</scope>
    </dependency>
</dependencies>
<build>
    <finalName>${project.artifactId}</finalName>
</build>
```

**SpringBootServletInitializer**

```
public class ServletInitializer extends SpringBootServletInitializer {

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        return application.sources(EurekaClientApplication.class);
    }

}
```

**weblogic.xml**

```
<?xml version="1.0" encoding="UTF-8"?>
<weblogic-web-app
        xmlns="http://xmlns.oracle.com/weblogic/weblogic-web-app"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://xmlns.oracle.com/weblogic/weblogic-web-app
                            http://xmlns.oracle.com/weblogic/weblogic-web-app/1.9/weblogic-web-app.xsd">
    <context-root>/client</context-root>
    <container-descriptor>
        <prefer-application-packages>
            <package-name>net.minidev.json.*</package-name>
            <package-name>com.jayway.*</package-name>
            <package-name>org.slf4j.*</package-name>
            <package-name>com.sun.jersey.*</package-name>
            <package-name>org.springframework.*</package-name>
            <package-name>aj.org.objectweb.*</package-name>
            <package-name>antlr.*</package-name>
            <package-name>antlr.ASdebug.*</package-name>
            <package-name>antlr.actions.cpp.*</package-name>
            <package-name>antlr.actions.csharp.*</package-name>
            <package-name>antlr.actions.java.*</package-name>
            <package-name>antlr.actions.python.*</package-name>
            <package-name>antlr.build.*</package-name>
            <package-name>antlr.collections.*</package-name>
            <package-name>antlr.collections.impl.*</package-name>
            <package-name>antlr.debug.*</package-name>
            <package-name>antlr.debug.misc.*</package-name>
            <package-name>antlr.preprocessor.*</package-name>
            <package-name>com.ctc.wstx.*</package-name>
            <package-name>com.fasterxml.classmate.*</package-name>
            <package-name>com.fasterxml.jackson.*</package-name>
            <package-name>com.google.common.*</package-name>
            <package-name>com.google.thirdparty.*</package-name>
            <package-name>com.sun.research.*</package-name>
            <package-name>com.sun.ws.*</package-name>
            <package-name>javax.annotation.*</package-name>
            <package-name>javax.annotation.security.*</package-name>
            <package-name>javax.annotation.sql.*</package-name>
            <package-name>javax.inject.*</package-name>
            <package-name>javax.validation.*</package-name>
            <package-name>javax.validation.bootstrap.*</package-name>
            <package-name>javax.validation.constraints.*</package-name>
            <package-name>javax.validation.constraintvalidation.*</package-name>
            <package-name>javax.validation.executable.*</package-name>
            <package-name>javax.validation.groups.*</package-name>
            <package-name>javax.validation.metadata.*</package-name>
            <package-name>javax.validation.spi.*</package-name>
<!--            <package-name>javax.ws.rs.*</package-name>-->
            <package-name>jersey.repackaged.org.*</package-name>
            <package-name>org.antlr.runtime.*</package-name>
            <package-name>org.aopalliance.aop.*</package-name>
            <package-name>org.aopalliance.intercept.*</package-name>
            <package-name>org.apache.commons.*</package-name>
            <package-name>org.bouncycastle.*</package-name>
            <package-name>org.bouncycastle.asn1.*</package-name>
            <package-name>org.bouncycastle.cert.*</package-name>
            <package-name>org.bouncycastle.cms.*</package-name>
            <package-name>org.bouncycastle.crypto.*</package-name>
            <package-name>org.bouncycastle.dvcs.*</package-name>
            <package-name>org.bouncycastle.eac.*</package-name>
            <package-name>org.bouncycastle.i18n.*</package-name>
            <package-name>org.bouncycastle.jcajce.*</package-name>
            <package-name>org.bouncycastle.jce.*</package-name>
            <package-name>org.bouncycastle.math.*</package-name>
            <package-name>org.bouncycastle.mozilla.*</package-name>
            <package-name>org.bouncycastle.openssl.*</package-name>
            <package-name>org.bouncycastle.operator.*</package-name>
            <package-name>org.bouncycastle.pkcs.*</package-name>
            <package-name>org.bouncycastle.pkix.*</package-name>
            <package-name>org.bouncycastle.pqc.*</package-name>
            <package-name>org.bouncycastle.tsp.*</package-name>
            <package-name>org.bouncycastle.util.*</package-name>
            <package-name>org.bouncycastle.voms.*</package-name>
            <package-name>org.bouncycastle.x509.*</package-name>
            <package-name>org.codehaus.jettison.*</package-name>
            <package-name>org.codehaus.stax2.*</package-name>
            <package-name>org.hibernate.validator.*</package-name>
            <package-name>org.jboss.logging.*</package-name>
            <package-name>org.joda.time.*</package-name>
        </prefer-application-packages>
    </container-descriptor>
</weblogic-web-app>
```

**jsr311-api-1.1.1.jar**

```
将jsr311-api-1.1.1.jar放入WebLogic的公共模块目录中，具体位置为：
${WEBLOGIC_HOME}/Middleware/wlserver/modules/
```

# \[Spring Boot 2.7.3 JDK17 Docker] 打包镜像

### Dockerfile (path: src/main/docker)

```
# 根据平台选择
FROM eclipse-temurin:17-jre-focal as builder
#FROM arm64v8/eclipse-temurin:17-jre-focal as builder
#FROM amd64/eclipse-temurin:17-jre-focal as builder

WORKDIR application

ARG JAR_FILE=target/*.jar

COPY ${JAR_FILE} application.jar

RUN java -Djarmode=layertools -jar application.jar extract

# 根据平台选择
FROM eclipse-temurin:17-jre-focal
#FROM arm64v8/eclipse-temurin:17-jre-focal
#FROM amd64/eclipse-temurin:17-jre-focal

WORKDIR application

COPY --from=builder application/dependencies/ ./

COPY --from=builder application/spring-boot-loader/ ./

COPY --from=builder application/snapshot-dependencies/ ./

COPY --from=builder application/application/ ./

ENV TZ="Asia/Shanghai"

RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

ENV JVM_OPTS=""

ENV JAVA_OPTS=""

ENTRYPOINT ["sh","-c","java $JVM_OPTS $JAVA_OPTS org.springframework.boot.loader.JarLauncher"]

```

### 使用

```
mvn clean package
mvn spring-boot:build-image
```

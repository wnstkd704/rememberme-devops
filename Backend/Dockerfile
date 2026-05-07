FROM maven:3.9.15-eclipse-temurin-21-alpine AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN [ "mvn", "package", "-DskipTests" ]
FROM eclipse-temurin:21-jre-alpine-3.22
LABEL maintainer="junsang"
LABEL version="1.0"
ENV TZ=Asia/Seoul
ARG BUILD_PROFILE=local
ARG BUILD_PORT=8088
ENV APP_PROFILE=${BUILD_PROFILE}
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE ${BUILD_PORT}
ENTRYPOINT ["sh", "-c", "java -jar app.jar --spring.profiles.active=${APP_PROFILE}"]
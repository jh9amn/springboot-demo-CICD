# Spring Boot Demo – Task REST API

Java 17+ and Maven required.

    mvn spring-boot:run

Endpoints (`/api/tasks`): GET, GET /{id}, POST, PUT /{id}, DELETE /{id}

    curl -X POST localhost:8080/api/tasks -H "Content-Type: application/json" -d '{"title":"Learn Spring Boot"}'
    curl localhost:8080/api/tasks

H2 console: http://localhost:8080/h2-console (JDBC URL: jdbc:h2:mem:demodb)


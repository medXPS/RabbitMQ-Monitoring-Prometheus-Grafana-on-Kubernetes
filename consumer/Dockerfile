FROM openjdk:17
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE}  consumer.jar
ENTRYPOINT ["java","-jar","/consumer.jar"]
EXPOSE 8080
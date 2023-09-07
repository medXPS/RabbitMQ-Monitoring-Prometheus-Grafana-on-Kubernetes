FROM openjdk:17
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE}  producer.jar
ENTRYPOINT ["java","-jar","/producer.jar"]
EXPOSE 8080
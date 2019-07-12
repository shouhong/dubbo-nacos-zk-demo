FROM openjdk:8-jre
VOLUME /tmp
COPY target/*.jar app.jar
COPY wait-for-it.sh /wait-for-it.sh
RUN chmod +x /wait-for-it.sh
RUN sh -c 'touch /app.jar'
CMD ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]

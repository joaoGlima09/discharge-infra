FROM eclipse-temurin:21-jdk-alpine

RUN apk add --no-cache git

WORKDIR /app

RUN git clone https://github.com/Grupo-5-High-Five/discharge-backend.git .

CMD java -jar target/discharge-1.0-SNAPSHOT-jar-with-dependencies.jar

EXPOSE 5555
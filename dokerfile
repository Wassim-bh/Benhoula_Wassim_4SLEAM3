FROM openjdk: 17.0.1-jdk-slim
WORKDIR /app
COPY target/*-jan /app-jar
EXPOSE 8089
ENTRYPOINT ['java", "-jar", "/app-jar"']

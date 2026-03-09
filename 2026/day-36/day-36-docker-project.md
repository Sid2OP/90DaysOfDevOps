Day 36 – Docker Project: Dockerize a Full Application

Docker file
```docker
FROM eclipse-temurin:21-jdk-jammy
WORKDIR /app
COPY . .
RUN apt-get update && apt-get install -y maven
RUN mvn clean package -DskipTests
EXPOSE 8080
ENTRYPOINT ["sh", "-c", "java -jar target/*.jar"]

```

docker-compose-file
```
services:
  mysql:
    image: mysql:8.0
    container_name: bankapp-mysql
    environment:
      MYSQL_ROOT_PASSWORD: Test@123
      MYSQL_DATABASE: bankappdb
    ports:
      - "3307:3306"
    volumes:
      - mysql-data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    networks:
      - bankapp-net

  ollama:
    image: ollama/ollama
    container_name: bankapp-ollama
    ports:
      - "11434:11434"
    volumes:
      - ollama-data:/root/.ollama
    networks:
      - bankapp-net

  bankapp:
    build: .
    container_name: bankapp
    ports:
      - "8080:8080"
    environment:
      MYSQL_HOST: mysql
      MYSQL_PORT: 3306
      MYSQL_DATABASE: bankappdb
      MYSQL_USER: root
      MYSQL_PASSWORD: Test@123
      OLLAMA_URL: http://ollama:11434
    depends_on:
      mysql:
        condition: service_healthy
    networks:
      - bankapp-net

volumes:
  mysql-data:
  ollama-data:

networks:
  bankapp-net:
    driver: bridge

```

![image.png](attachment:019470fb-b339-4455-9deb-519af660b7cb:image.png)
-----------------------------

![image.png](attachment:61c1b77b-7d42-4af2-afed-2753068261e8:image.png)

-----------------------------

![image.png](attachment:19128adc-d085-451b-85f1-d847c6be1c91:image.png)

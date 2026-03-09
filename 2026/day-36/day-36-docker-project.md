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

<img width="1918" height="1017" alt="image" src="https://github.com/user-attachments/assets/1e372d47-0695-46e2-bb18-d902355989d8" />

-----------------------------

<img width="1898" height="903" alt="image" src="https://github.com/user-attachments/assets/97a3b681-e151-41aa-ab15-3e02afae2794" />

-----------------------------

<img width="1898" height="846" alt="image" src="https://github.com/user-attachments/assets/9a7c1b34-797e-469a-a2de-83675d119dd4" />


# Лабораторная работа №4

## Постановка задачи

**Тема**: Реализация архитектуры на основе сервисов (микросервисной архитектуры)

**Цель работы**: Получить опыт работы организации взаимодействия сервисов с использованием контейнеров Docker

**Ожидаемые результаты**:

1.	Согласно диаграмме контейнеров реализовать в отдельности каждый контейнер и настроить между ними взаимодействие. Выделить минимум 3 контейнера: клиентская часть, серверная часть и БД. Запустить контейнеры, показать работоспособность приложения, состоящего из взаимодействующих сервисов (запускать можно локально или на удаленной машине). (4 балла)
2.	Настроить непрерывную интеграцию (сборку приложения и создание docker-образов). (2 балла)
3.  Разработать интеграционные тесты и включить их в процесс непрерывной интеграции (можно подключить тесты, ранее созданные в Postman). (2 балла)
4.  Настроить непрерывное развертывание (развертывание реализовать можно публикацией образов на https://hub.docker.com/). (2 балла)

### Dockerfile контейнеров service и worker

Service - основное API и доменная логика

Worker - закрытый сервис, реализующий регулярные события

Проекты написаны на .Net, используется шаблонный multi-stage build с включенными тестами (без пройденных тестов запуск невозможен)

```yml
### build
FROM mcr.microsoft.com/dotnet/sdk:9.0 as build
WORKDIR /sln
COPY ./*.sln ./src/*/*.csproj ./
RUN mkdir src
RUN for file in $(ls *.csproj); do mkdir -p src/${file%.*}/ && mv $file src/${file%.*}/; done

RUN dotnet restore "BabloBudget.sln"
COPY . .
RUN dotnet build "BabloBudget.sln" -c Release --no-restore

## tests
FROM mcr.microsoft.com/dotnet/sdk:9.0 as tests
WORKDIR /tests
COPY --from=build /sln/src/Tests .
ENTRYPOINT ["dotnet", "test", "-c", "Release", "--no-build"]

### publish
FROM build as publish
RUN dotnet publish "./src/BabloBudget.Api/BabloBudget.Api.csproj" -c Release --output /dist/services --no-restore
RUN dotnet publish "./src/BabloBudget.Worker/BabloBudget.Worker.csproj" -c Release --output /dist/worker --no-restore

### services
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS services
EXPOSE 8018
ENV ASPNETCORE_URLS=http://+:8018
WORKDIR /app
COPY --from=publish /dist/services .
ENTRYPOINT ["dotnet", "BabloBudget.Api.dll"]

### worker
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS worker
EXPOSE 8019
ENV ASPNETCORE_URLS=http://+:8019
WORKDIR /app
COPY --from=publish /dist/worker .
ENTRYPOINT ["dotnet", "BabloBudget.Worker.dll"]
```

### Dockerfile контейнера frontend

Фронт написан на Vite, используется стандартный шаблон через nginx

```yml
# ---------- build stage ----------
FROM node:20-alpine AS build

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# ---------- runtime stage ----------
FROM nginx:alpine

COPY --from=build /app/dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

Для БД используется готовый контейнер Posrgres 16.

### CI/CD пайплайны GitHub Actions для тестирования и сборки контейнеров

**Пайплайн на тестирование при мердже в main**

```yml
name: Unit Tests

on: 
  pull_request:
  workflow_dispatch:

env:
  DOTNET_VERSION: "9.0.x"

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
        
      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with: 
          dotnet-version: '9.0.x'
      
      - name: Restore
        run: dotnet restore ./BabloBudget.sln
        
      - name: Build
        run: dotnet build ./BabloBudget.sln --configuration Release --no-restore
        
      - name: Test
        run: dotnet test ./BabloBudget.sln --configuration Release --no-restore --no-build
```

**Пайплайн на сборку и публикацию на dockerhub**

```yml
name: Build and Publish Docker Images

on:
  push:
    branches: [ "main" ]
  workflow_dispatch:

env:
  IMAGE_NAME: likeitbro/bablobudget

jobs:
  docker:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      # ---- SERVICE ----
      - name: Build and push service
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile.service
          push: true
          tags: ${{ env.IMAGE_NAME }}:service

      # ---- WORKER ----
      - name: Build and push worker
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile.worker
          push: true
          tags: ${{ env.IMAGE_NAME }}:worker

      # ---- FRONTEND ----
      - name: Build and push frontend
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile.frontend
          push: true
          tags: ${{ env.IMAGE_NAME }}:frontend
```

### docker-compose.yml для проекта

По этому правилу с использованием параметров .env запускаются контейнеры на VDS. На VDS открыты порты 8018 для API и 3001 для фронта.

```yml
version: "3.9"

services:
  bablobudget-services:
    image: likeitbro/bablobudget:services
    restart: always
    environment:
      ASPNETCORE_ENVIRONMENT: Production
      ConnectionStrings__DefaultConnection: "Host=db;Database=${POSTGRES_DB};Username=postgres;Password=${POSTGRES_PASSWORD};"
    depends_on:
      - db
    networks:
      - bablobudgetnetwork
    ports:
      - "8018:8018"

  bablobudget-worker:
    image: likeitbro/bablobudget:worker
    restart: always
    environment:
      ASPNETCORE_ENVIRONMENT: Production
      ConnectionStrings__DefaultConnection: "Host=db;Database=${POSTGRES_DB};Username=postgres;Password=${POSTGRES_PASSWORD};"
    depends_on:
      - bablobudget-services
    networks:
      - bablobudgetnetwork

  frontend:
    image: likeitbro/bablobudget:frontend
    restart: always
    depends_on:
      - bablobudget-services
    networks:
      - bablobudgetnetwork
    ports:
      - "3001:80"
    environment:
      VITE_BACKEND_URL: ${VITE_BACKEND_URL}

  db:
    image: postgres:16
    container_name: bablobudgetdb
    restart: always
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    networks:
      - bablobudgetnetwork
    volumes:
      - database:/var/lib/postgresql/data
    ports:
      - "5433:5432"

volumes:
  database:

networks:
  bablobudgetnetwork:
    driver: bridge
```
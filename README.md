# Discharge-infra

## Índice

1. [Descrição do Projeto](#descrição-do-projeto)
2. [Tecnologias Utilizadas](#tecnologias-utilizadas)
3. [Estrutura do Repositório](#estrutura-do-repositório)
4. [Configurações das Imagens Dockerfiles](#configurações-das-imagens-dockerfiles)
5. [Configuração do Docker Compose](#configuração-do-docker-compose)
6. [Configuração do CloudFormation](#configuração-do-cloudformation)
7. [Configuração do Shell Script](#configuração-do-shell-script)

## Descrição do Projeto

Este repositório contém as configurações para implantar a infraestrutura utilizando Docker Compose, AWS CloudFormation e um esquema SQL inicial.

## Tecnologias Utilizadas

- [Docker](https://www.docker.com/)
- [MySQL](https://www.mysql.com/)
- [Shell Script](https://www.devmedia.com.br/introducao-ao-shell-script-no-linux/25778)
- [AWS CloudFormation](https://aws.amazon.com/cloudformation/)

## Estrutura do Repositório

```
├── docker
|   └── backend
|       └── Dockerfile
|   └── frontend
|       └── Dockerfile
├── docker-compose.yml
├── .env.example
├── init.sql
├── install.sh
├── main.yaml
```

## Configurações das Imagens Dockerfiles

Os Dockerfiles estão sendo utilizados para criar a imagens de Java e Node que estão armazenadas em um DockerHub para que seja possível a criação de contianers com imagens personalizadas de acordo com o projeto Discharge

zapss/discharge-jdk:1.0

```
FROM eclipse-temurin:21-jdk-alpine

RUN apk add --no-cache git

WORKDIR /app

RUN git clone https://github.com/Grupo-5-High-Five/discharge-backend.git .

CMD java -jar target/discharge-1.0-SNAPSHOT-jar-with-dependencies.jar

EXPOSE 5555
```

zapss/discharge-nodejs:1.0

```
FROM node:20.18.0-alpine

RUN apk add --no-cache git

WORKDIR /app

RUN git clone https://github.com/Grupo-5-High-Five/discharge-web.git .

RUN npm install

CMD node --watch app.js

EXPOSE 3333
```

## Configuração do Docker Compose

O Docker compose está sendo utilizado para criação de 3 containers.

### Configuração de Variáveis de Ambiente

Antes de executar o Docker Compose, é necessário configurar as variáveis de ambiente:

1. Copie o arquivo `.env.example` para `.env`:
```bash
cp .env.example .env
```

2. Preencha o arquivo `.env` com os valores adequados:
```bash
# MySQL Database Configuration
MYSQL_ROOT_PASSWORD=sua_senha_root
MYSQL_DATABASE=discharge
MYSQL_USER=user
MYSQL_PASSWORD=sua_senha

# Backend Database Configuration
BACKEND_DB_USER=root
BACKEND_DB_PASSWORD=sua_senha_root

# Frontend Database Configuration
FRONTEND_DB_HOST=db
FRONTEND_DB_DATABASE=discharge
FRONTEND_DB_USER=user
FRONTEND_DB_PASSWORD=sua_senha
FRONTEND_DB_PORT=3306

# Frontend Application Configuration
APP_PORT=80
APP_HOST=localhost
AMBIENTE_PROCESSO=desenvolvimento

# Email Configuration
MEU_EMAIL=seu_email@example.com
MINHA_SENHA=sua_senha_email

# AWS Configuration
AWS_ACCESS_KEY_ID=sua_access_key
AWS_SECRET_ACCESS_KEY=sua_secret_key
AWS_SESSION_TOKEN=seu_session_token
TOKEN=seu_token
```

### Containers

mysql_container

Criando o banco de dados a partir do arquivo init.sql (que contém o esquema do banco de dados MySQL) e o usuário que realiza as manipulações dentro do banco de dados. As credenciais são carregadas a partir das variáveis de ambiente definidas no arquivo `.env`.

```
  db:
    image: mysql:8.0
    container_name: mysql_container
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    restart: always
    healthcheck:
      test: mysql -u root -p$${MYSQL_ROOT_PASSWORD} -h 127.0.0.1 -e 'SHOW DATABASES;'
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
```

java_container

Realiza a execução da aplicação em Java, utilizando o usuário do banco de dados criado pelo mysql_container - Acessando a AWS para leitura de arquivos dentro do S3 e às inserindo no banco de dados. As credenciais do banco de dados e da AWS são carregadas a partir das variáveis de ambiente definidas no arquivo `.env`.

```
  backend:
    image: zapss/discharge-jdk:latest
    container_name: java_container
    environment:
      # Acesso ao Container DB
      DB_URL: jdbc:mysql://db/${MYSQL_DATABASE}
      DB_USER: ${BACKEND_DB_USER}
      DB_PASSWORD: ${BACKEND_DB_PASSWORD}
      # Acesso ao S3
      AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}
      AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
      AWS_SESSION_TOKEN: ${AWS_SESSION_TOKEN}
      TOKEN: ${TOKEN}
    ports:
      - "5555:5555"
    depends_on:
      db:
        condition: service_healthy
```

node_container

Roda a aplicação Web com o NodeJS e visualiza informações vindas do banco de dados. Todas as configurações de conexão ao banco, porta da aplicação e credenciais de email são carregadas a partir das variáveis de ambiente definidas no arquivo `.env`.

```
  frontend:
    image: zapss/discharge-nodejs:latest
    container_name: node_container
    environment:
      AMBIENTE_PROCESSO: ${AMBIENTE_PROCESSO}
      DB_HOST: ${FRONTEND_DB_HOST}
      DB_DATABASE: ${FRONTEND_DB_DATABASE}
      DB_USER: ${FRONTEND_DB_USER}
      DB_PASSWORD: ${FRONTEND_DB_PASSWORD}
      DB_PORT: ${FRONTEND_DB_PORT}
      APP_PORT: ${APP_PORT}
      APP_HOST: ${APP_HOST}
      MEU_EMAIL: ${MEU_EMAIL}
      MINHA_SENHA: ${MINHA_SENHA}
    restart: always
    ports:
      - "80:80"
    depends_on:
      db:
        condition: service_healthy
```

**⚠️ Importante:** O arquivo `.env` contém informações sensíveis e **não deve ser commitado** no repositório. Apenas o arquivo `.env.example` (com valores vazios) deve ser versionado. Certifique-se de que o `.env` está listado no `.gitignore`.

## Configuração do CloudFormation

No arquivo main.yaml é possível visualizar as configurações necessárias para que o projeto seja executado na AWS. Dessa forma, é automatizado a criação da EC2 e suas demais configurações. \*Necessário apenas a criação de uma pilha(stack)

EC2 (Definições de Hardware e Acesso)

Grupos de Segurança (Liberação de portas do projeto)

VPC (Com as demais configurações para o seu acesso)

## Configuração do Shell Script

No arquivo install.sh, está as configurações essenciais para o Ubuntu Server que está instalado na EC2 criada pelo CloudFormattion. O Script de instalação, atualiza os pacotes do Linux, baixa o Docker-Compose e inícia todos os containers de forma automatizada, sendo necessário a instalação de um arquivo.

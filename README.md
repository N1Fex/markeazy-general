# Markeazy
Monorepo (with submodules) for the Markeazy marketplace platform.
This repository ties together:
- Backend — Spring Boot (Java 17) service with security, PostgreSQL persistence and Elasticsearch integration. 
- Frontend — React app (Create React App, React 19 + React Router + MUI). 
- Infrastructure — Docker Compose for local development: backend, frontend, PostgreSQL, pgAdmin, and Elasticsearch. 
The goal is to provide an end-to-end local dev / demo environment for the marketplace: API, DB, search, and UI — all up with a single ```docker compose up```.

# Repository structure
```
markeazy-general/
├─ backend/        # Git submodule -> https://github.com/N1Fex/markeazy-backend
├─ frontend/       # Git submodule -> https://github.com/N1Fex/markeazy-frontend
├─ docker-compose.yml
├─ .gitmodules
├─ .gitignore
└─ (your local) .env
```

## Backend (```backend/```)
Spring Boot 3.x service (```Java 17```) that exposes the API (base path ```/api/v1```) and talks to:
- **PostgreSQL** for persistence
- **Elasticsearch** for search / discovery
- **JWT** auth (JJWT)
- **Spring Security**, **Spring Data JPA**, **Spring Web**, **Spring Data Elasticsearch**
Dependencies in ```pom.xml``` include:
```
spring-boot-starter-web
spring-boot-starter-security
spring-boot-starter-data-jpa
spring-boot-starter-data-elasticsearch
jjwt (api / impl / jackson)
postgresql driver
lombok
mapstruct
```
The backend Dockerfile builds and runs the service using Maven inside a ```temurin-17``` image and starts the app with ```mvn spring-boot:run```. 
The backend container runs with ```SPRING_PROFILES_ACTIVE=prod``` and gets ```ELASTICSEARCH_URL``` from environment. 

## Frontend (```frontend/```)
React SPA bootstrapped with Create React App. It uses:
- React 19
- React Router DOM 7
- Material UI (```@mui/material```, ```@mui/icons-material```)
- Axios for API calls
- ```jwt-decode``` for client-side auth token handling
The Dockerfile builds the React app in a Node 20 Alpine container. It accepts ```REACT_APP_BACKEND_URL``` as a build arg / env so the UI knows where the API lives, e.g. ```http://localhost:8080/api/v1```. 
By default CRA dev server runs on ```FRONTEND_PORT``` (typically ```3000```).

## Services
```docker-compose.yml``` defines the full local stack:
- **backend** (container name ```spring-backend```)
- **frontend** (container name ```react-frontend```)
- **db**: PostgreSQL 16 Alpine
- **pgadmin**: pgAdmin 4 (web UI for Postgres) exposed on ```localhost:5050```
- **elasticsearch**: single-node Elasticsearch 8.x, security disabled, exposed on ```localhost:9200```
All services share configuration through a local ```.env``` file.

# Prerequisites
- Git (with submodule support)
- Docker + Docker Compose
- Java/Maven and Node are not strictly required on your host if you use only Docker, because both backend and frontend are built inside containers.

## Clone the repo (with submodules)
This repo uses Git submodules for ```backend/``` and ```frontend/```. 
Fresh clone:
```
git clone --recurse-submodules https://github.com/N1Fex/markeazy-general.git
cd markeazy-general
```
If you already cloned without ```--recurse-submodules```:
```
git submodule update --init --recursive
```

## 2. Create ```.env```
In the project root (```markeazy-general/.env```) create environment variables used by ```docker-compose.yml```.

Below is a sane local dev example.
You can change ports / credentials if something is already in use on your machine.
```
# === Ports ===
BACKEND_PORT=8080
FRONTEND_PORT=3000
DB_PORT=5432

# === Postgres ===
POSTGRES_USER=markeazy
POSTGRES_PASSWORD=markeazy
POSTGRES_DB_NAME=markeazy

# === pgAdmin ===
PGADMIN_DEFAULT_EMAIL=admin@admin.com
PGADMIN_DEFAULT_PASSWORD=admin

# === Search ===
ELASTICSEARCH_URL=http://elastic-search:9200

# (any other Spring/React secrets go here, e.g. JWT signing keys, etc.)
```
How these are used:
- ```backend``` container exposes ```${BACKEND_PORT}``` and reads ```ELASTICSEARCH_URL```.
- ```frontend``` container exposes ```${FRONTEND_PORT}``` and is built with ```REACT_APP_BACKEND_URL=http://localhost:${BACKEND_PORT}/api/v1```.
- ```db``` runs Postgres 16 on ```${DB_PORT}``` and seeds database ```${POSTGRES_DB_NAME}``` with ```${POSTGRES_USER}``` / ```${POSTGRES_PASSWORD}```.
- ```pgadmin``` lets you inspect Postgres at ```http://localhost:5050``` using the pgAdmin credentials above.
- ```elasticsearch``` runs on ```http://localhost:9200``` as a single-node dev cluster

№№ 3. Run the full stack
From the root of ```markeazy-general```:
```
docker compose up --build
```
What happens:
- **Elasticsearch** starts first (```elasticsearch``` service).
- **PostgreSQL** starts with a persistent volume ```db-data:```.
- **Backend** builds via Maven in its own Dockerfile and boots Spring Boot with ```SPRING_PROFILES_ACTIVE=prod```.
- **Frontend** installs deps, builds the React app, and starts the dev server.
- **pgAdmin** starts so you can inspect the DB via browser.

All containers are defined in ```docker-compose.yml```:
```
name: markeazy

services:
  backend:
    container_name: spring-backend
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - ${BACKEND_PORT}:${BACKEND_PORT}
    depends_on:
      - db
      - elasticsearch
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - ELASTICSEARCH_URL=${ELASTICSEARCH_URL}
    env_file:
      - .env

  frontend:
    container_name: react-frontend
    build:
      context: ./frontend
      dockerfile: Dockerfile
      args:
        REACT_APP_BACKEND_URL: http://localhost:${BACKEND_PORT}/api/v1
    depends_on:
      - backend
    ports:
      - ${FRONTEND_PORT}:${FRONTEND_PORT}

  db:
    container_name: postgres
    image: postgres:16.10-alpine
    env_file:
      - .env
    environment:
      - POSTGRES_DB=${POSTGRES_DB_NAME}
    ports:
      - ${DB_PORT}:5432
    healthcheck:
      test: ["CMD", "pg_isready -U ${POSTGRES_USER} -d $${POSTGRES_DB_NAME} -p ${DB_PORT}"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    volumes:
      - db-data:/var/lib/postgresql/data

  pgadmin:
    container_name: pgadmin4
    image: dpage/pgadmin4:9.9
    ports:
      - "5050:80"
    env_file:
      - .env

  elasticsearch:
    container_name: elastic-search
    image: elastic/elasticsearch:8.18.8
    ports:
      - "9200:9200"
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms128m -Xmx128m
    volumes:
      - es-data:/usr/share/elasticsearch/data

volumes:
  es-data:
  db-data:
```
(Formatted from the repo’s docker-compose.yml.)

## 4. Common developer tasks
### Rebuild after code changes

If you change backend Java code or frontend React code and want containers rebuilt:
```
docker compose build
docker compose up
```
Stopping everything
```
docker compose down
```

To also remove the persistent volumes (***destroys DB/search data***):
```
docker compose down -v
```

## 5. Tech stack summary
**Backend**
- Java 17 / Spring Boot 3.x
- Spring Security + JWT (JJWT)
- Spring Data JPA (PostgreSQL)
- Spring Data Elasticsearch (search)
- MapStruct for DTO ↔ entity mapping
- Lombok to reduce boilerplate
- Maven build / packaging

**Frontend**
- React 19 (Create React App)
- React Router DOM 7
- Material UI (```@mui/material```, ```@mui/icons-material```)
- Axios for HTTP
- ```jwt-decode``` for auth token handling

**Infrastructure**

Docker Compose orchestrating:
- Spring backend (```spring-backend```)
- React frontend (```react-frontend```)
- PostgreSQL 16 (persistent volume ```db-data```)
- pgAdmin 4 (web DB admin at ```:5050```)
- Elasticsearch 8 (persistent volume ```es-data```, security disabled for local dev)

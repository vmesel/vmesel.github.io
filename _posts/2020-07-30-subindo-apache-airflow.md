---
published: true
title: Subindo um Apache Airflow para a Amazon utilizando o ECS
layout: post
tags:
  - airflow
  - ecs
  - aws
  - python
categories:
  - airflow
  - ecs
  - aws
  - python
permalink: tutorial-airflow-no-ecs
---
Hoje em dia, cada vez menos dependemos de aplicações hospedadas em servidores específicos, podemos roda-las conteinerizadas e em ambientes serverless, assim economizamos tempo de gerenciamento de maquina e focamos realmente em entregar valor!

O Airflow não é exceção, ele é um software que pode ser hospedado no AWS ECS (Elastic Container Service) via docker e nesse tutorial eu vou lhes mostrar como fazer isso!

### Montando a imagem Docker

A imagem Docker do airflow, para nossa sorte, já foi desenvolvida pelo usuário [puckel](https://github.com/puckel), mas ela precisa de algumas modificações para rodarmos tudo o que precisamos.

```
FROM puckel/docker-airflow:1.10.7

USER root

COPY /dags ./dags

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
COPY requirements.txt /requirements.txt
RUN pip install -r /requirements.txt

# Add directory in which pip installs to PATH
ENV PATH="/usr/local/airflow/.local/bin:${PATH}"

USER airflow
COPY --chown=airflow:airflow airflow.cfg /usr/local/airflow/airflow.cfg

ENTRYPOINT ["/entrypoint.sh"]

EXPOSE 8080
EXPOSE 8793
EXPOSE 5555
```

Como vemos acima, para essa imagem rodar, precisamos fazer algumas coisas:

 - Criar uma pasta de DAGs onde elas serão colocadas para leitura e execução no Airflow
 - Criar um arquivo de `entrypoint.sh` onde ficam algumas informações de entrada de sistemas
 - Criar um arquivo de `requirements.txt` onde ficarão todas as bibliotecas extras e necessárias para a execução dos softwares
 - Criar um arquivo de `airflow.cfg` onde fique todas as configurações do Airflow
 
 
 ### Criando a pasta de DAGs
 
 
 Esta é a parte mais tranquila de fazer, basta você fazer um:
 
 ```
 mkdir dags
 ```
 
 E pronto! xD
 
 
 ### Arquivo de Chamada de Processos - entrypoint.sh
 
 
 O arquivo de entrypoint servirá como uma base para podermos iniciar diferentes containers com diversas funções com a mesma imagem, assim conseguimos atribuir a função corretamente.
 
 ```bash
 #!/usr/bin/env bash

: "${REDIS_HOST:="redis"}"
: "${REDIS_PORT:="6379"}"
: "${REDIS_PASSWORD:=""}"

: "${POSTGRES_HOST:=""}"
: "${POSTGRES_PORT:="5432"}"
: "${POSTGRES_USER:=""}"
: "${POSTGRES_PASSWORD:=""}"
: "${POSTGRES_DB:=""}"
: "${MONGO_DB_ATLAS_URI:=""}"

# Defaults and back-compat
: "${AIRFLOW_HOME:="/usr/local/airflow"}"
: "${AIRFLOW__CORE__FERNET_KEY:=${FERNET_KEY:=$(python -c "from cryptography.fernet import Fernet; FERNET_KEY = Fernet.generate_key().decode(); print(FERNET_KEY)")}}"
: "${AIRFLOW__CORE__EXECUTOR:=${EXECUTOR:-Celery}Executor}"

if [ -n "$REDIS_PASSWORD" ]; then
    REDIS_PREFIX=:${REDIS_PASSWORD}@
else
    REDIS_PREFIX=
fi

AIRFLOW__CELERY__BROKER_URL="redis://$REDIS_PREFIX$REDIS_HOST:$REDIS_PORT/1"
AIRFLOW__CORE__SQL_ALCHEMY_CONN="postgresql+psycopg2://$POSTGRES_USER:$POSTGRES_PASSWORD@$POSTGRES_HOST:$POSTGRES_PORT/$POSTGRES_DB"
AIRFLOW__CELERY__RESULT_BACKEND="db+postgresql://$POSTGRES_USER:$POSTGRES_PASSWORD@$POSTGRES_HOST:$POSTGRES_PORT/$POSTGRES_DB"


export \
  AIRFLOW_HOME \
  AIRFLOW__CELERY__BROKER_URL \
  AIRFLOW__CELERY__RESULT_BACKEND \
  AIRFLOW__CORE__EXECUTOR \
  AIRFLOW__CORE__FERNET_KEY \
  AIRFLOW__CORE__LOAD_EXAMPLES \
  AIRFLOW__CORE__SQL_ALCHEMY_CONN \


# Load DAGs examples (default: Yes)
if [[ -z "$AIRFLOW__CORE__LOAD_EXAMPLES" && "${LOAD_EX:=n}" == n ]]
then
  AIRFLOW__CORE__LOAD_EXAMPLES=False
fi


case "$1" in
  webserver)
    echo "INITDB"
    airflow initdb
    echo "CREATING USER"
    airflow create_user -r Admin -u USUARIO -e EMAIL@EMAIL.com -f Admin -l descricao -p SENHA1234
    echo "DEPLOY DONE"
    exec airflow webserver
    ;;
  scheduler|flower|version)
    exec airflow "$@"
    ;;
  worker)
    exec airflow "$@"
    ;;
  *)
    # The command is something like bash, not an airflow subcommand. Just run it in the right environment.
    exec "$@"
    ;;
esac
 ```
 
Mais para frente, dentro do `docker-compose.yml`, você entenderá mais sobre a utilidade deste arquivo!

### Criando um arquivo de `requirements.txt` - requisitos das suas DAGs

Este arquivo também é fácil de ser criado! Ele é só o conjunto de libs que você quer que seja instalado dentro do seu Airflow, porém tome cuidado sempre com as versões das libs que você está usando e se ela já vem instalada previamente no container!

### Criando um arquivo de `airflow.cfg` - configurações do Airflow

O arquivo `airflow.cfg` é basicamente a configuração do seu Airflow. Existem algumas coisas que você pode escolher setar, como qual executor você vai querer usar, tipos de autenticação e muitas outras configurações.

Recomendo você pegar este arquivo originalmente do Airflow e configurar conforme suas necessidades!


## Docker-Compose, ou como a gente vai fazer o deploy de toda essa bodega

Nosso docker-compose é este:

```
version: '3'

services:
  redis:
    image: 'redis:5.0.3'
    command: redis-server

  postgres:
    image: postgres:10.4
    environment:
      - POSTGRES_USER=airflow
      - POSTGRES_PASSWORD=airflow
      - POSTGRES_DB=airflow
    volumes:
      - ./postgres_data:/var/lib/postgresql/data
    ports:
      - 5432:5432

  webserver:
    build: .
    restart: always
    build: .
    depends_on:
      - postgres
      - redis
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - POSTGRES_HOST=postgres
      - POSTGRES_PORT=5432
      - POSTGRES_USER=airflow
      - POSTGRES_PASSWORD=airflow
      - POSTGRES_DB=airflow
      - FERNET_KEY=${AIRFLOW_FERNET_KEY}
      - AIRFLOW_BASE_URL=http://localhost:8080
      - ENABLE_REMOTE_LOGGING=False
      - STAGE=dev
    ports:
        - "8080:8080"
    command: webserver
    healthcheck:
      test: ["CMD-SHELL", "[ -f /usr/local/airflow/airflow-webserver.pid ]"]
      interval: 30s
      timeout: 30s
      retries: 3

  flower:
    build: .
    restart: always
    depends_on:
      - redis
      - webserver
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - POSTGRES_HOST=postgres
      - POSTGRES_PORT=5432
      - POSTGRES_USER=airflow
      - POSTGRES_PASSWORD=airflow
      - POSTGRES_DB=airflow
      - STAGE=dev
    ports:
      - "5555:5555"
    command: flower

  scheduler:
    build: .
    restart: always
    depends_on:
      - webserver
    volumes:
      - ./dags:/usr/local/airflow/dags
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - POSTGRES_HOST=postgres
      - POSTGRES_PORT=5432
      - POSTGRES_USER=airflow
      - POSTGRES_PASSWORD=airflow
      - POSTGRES_DB=airflow
      - FERNET_KEY=${AIRFLOW_FERNET_KEY}
      - AIRFLOW_BASE_URL=http://localhost:8080
      - ENABLE_REMOTE_LOGGING=False
      - STAGE=dev
    command: scheduler

  worker:
    build: .
    restart: always
    depends_on:
      - webserver
      - scheduler
    volumes:
      - ./dags:/usr/local/airflow/dags
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - POSTGRES_HOST=postgres
      - POSTGRES_PORT=5432
      - POSTGRES_USER=airflow
      - POSTGRES_PASSWORD=airflow
      - POSTGRES_DB=airflow
      - FERNET_KEY=${AIRFLOW_FERNET_KEY}
      - AIRFLOW_BASE_URL=http://localhost:8080
      - ENABLE_REMOTE_LOGGING=False
      - STAGE=dev
    command: worker
```



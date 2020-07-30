---
published: false
---
## Subindo um Apache Airflow para a Amazon utilizando o ECS

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
 
 


# Control de versiones y CI/CD – Parte II

## 1. Introducción

Se parte de una aplicación Python desarrollada con **Flask**.  
El objetivo es que los cambios realizados en el repositorio de **GitHub**:

- Disparen la creación de una imagen **Docker**
- Actualicen la imagen en **DockerHub**
- Desplieguen automáticamente la aplicación en **AWS EC2**

Todo el proceso se realiza mediante **GitHub Actions**.

---

## 2. Conociendo la aplicación

Antes de comenzar se recomienda reiniciar los servicios:

```bash
sudo systemctl restart docker.service
sudo systemctl restart networking.service
2.1. Prueba en local
Para comprobar el funcionamiento de la aplicación:

python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
python src/app.py
Comprobación:

curl http://localhost:5000
curl http://localhost:5000/status
En producción se utiliza Gunicorn:

gunicorn --chdir src app:app --bind 0.0.0.0:5000

2.2. Creación de la imagen Docker
Dockerfile utilizado:

FROM python:3-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY src .
CMD gunicorn --bind 0.0.0.0:5000 app:app
Construcción y prueba:

docker build -t <usuariodockerhub>/galeria:latest .
docker run -d -p 80:5000 --name testgaleria <usuariodockerhub>/galeria

2.3. Prueba con Docker Compose
Fichero docker-compose.yml:

services:
  galeria:
    image: <usuariodockerhub>/galeria:latest
    ports:
      - 80:5000
    restart: always
Ejecución:

docker compose up -d
docker compose down

2.4. Subida de la imagen a DockerHub
docker push <usuariodockerhub>/galeria:latest

3. GitHub Actions: Build and Push
Se crea un workflow que construye y sube la imagen Docker automáticamente cuando hay cambios en src.

3.1. Secretos en GitHub
Se crean los siguientes secretos:

DOCKERHUB_USERNAME

DOCKERHUB_TOKEN

3.2. Workflow de construcción
name: Docker CI/CD

on:
  push:
    branches: [ main ]
    paths:
      - src/**

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/galeria:latest

4. Action: Despliegue en AWS
Se añade un segundo job que despliega la aplicación en una instancia EC2.

4.1. Secretos necesarios
AWS_USERNAME

AWS_HOSTNAME

AWS_PRIVATEKEY

4.2. Job de despliegue
Este job depende del build:

aws:
  needs: build
  runs-on: ubuntu-latest
  steps:
    - uses: appleboy/scp-action@v0.1.7
    - uses: appleboy/ssh-action@v1.0.3
El despliegue ejecuta:

docker compose down
docker compose up -d

5. Orden de ejecución en GitHub Actions
Los steps se ejecutan de forma secuencial

Los jobs se ejecutan en paralelo

Para definir dependencias se usa needs

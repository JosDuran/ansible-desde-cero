# Laboratorio 01: Entorno Ansible Automatizado con Docker

Este laboratorio despliega un entorno completo para Ansible (1 Nodo de Control con VS Code Web y 2 Nodos Administrados) de forma optimizada. El proceso reduce la configuración manual al mínimo, permitiéndote enfocarte en la administración.

---

## 1. Objetivos del Laboratorio

* Generar un par de claves SSH para la autenticación de Ansible.
* Desplegar un entorno multicontenedor utilizando Docker Compose y Dockerfiles.
* Validar la conexión SSH sin contraseña desde el nodo de control a los nodos administrados.

---

## 2. Preparación y Creación de Claves SSH

Ansible necesita autenticarse en los nodos administrados. Para hacerlo de forma segura y automatizada, primero generaremos las claves SSH en tu máquina local. 

Crea la carpeta de tu laboratorio e ingresa a ella:
```bash
mkdir lab-ansible && cd lab-ansible

ssh-keygen -t rsa -b 4096 -f ./ansible_key -q -N ""

## 3. Archivos de Configuración Docker
En la misma carpeta lab-ansible, crea los siguientes 4 archivos que automatizarán la instalación de SSH, la creación del usuario ansible y la distribución de las claves.

1. docker-compose.yaml
Este archivo orquesta el levantamiento de los tres servidores:

services:
  control-node:
    build:
      context: .
      dockerfile: Dockerfile.control
    container_name: ansible-control
    environment:
      - PASSWORD=ansible
    volumes:
      - ./workspace:/home/ansible/workspace
    ports:
      - "8443:8443"
    networks:
      - ansible-net
    restart: unless-stopped

  ubuntu-node:
    build:
      context: .
      dockerfile: Dockerfile.ubuntu
    container_name: ansible-ubuntu
    privileged: true
    cgroup: host
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    command: /lib/systemd/systemd
    networks:
      - ansible-net

  rocky-node:
    build:
      context: .
      dockerfile: Dockerfile.rocky
    container_name: ansible-rocky
    privileged: true
    cgroup: host
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    command: /usr/sbin/init
    networks:
      - ansible-net

networks:
  ansible-net:
    driver: bridge

2. Dockerfile.control
Construye el nodo de control. Copia ambas claves (pública y privada).

FROM ubuntu:22.04
ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y curl ansible openssh-client sudo nano iputils-ping
RUN curl -fsSL [https://code-server.dev/install.sh](https://code-server.dev/install.sh) | sh

RUN useradd -m -s /bin/bash ansible && \
    echo "ansible:ansible" | chpasswd && \
    usermod -aG sudo ansible && \
    echo "ansible ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

USER ansible
WORKDIR /home/ansible

# Inyectar claves SSH y deshabilitar StrictHostKeyChecking
RUN mkdir -p /home/ansible/.ssh
COPY --chown=ansible:ansible ansible_key /home/ansible/.ssh/id_rsa
COPY --chown=ansible:ansible ansible_key.pub /home/ansible/.ssh/id_rsa.pub
RUN chmod 600 /home/ansible/.ssh/id_rsa && \
    chmod 644 /home/ansible/.ssh/id_rsa.pub && \
    echo "Host *\n\tStrictHostKeyChecking no\n" > /home/ansible/.ssh/config && \
    chmod 600 /home/ansible/.ssh/config

EXPOSE 8443
CMD ["code-server", "--bind-addr", "0.0.0.0:8443", "--auth", "password", "/home/ansible/workspace"]

3. Dockerfile.ubuntu
Construye el nodo administrado Ubuntu. Copia solo la clave pública.

FROM geerlingguy/docker-ubuntu2204-ansible:latest

RUN apt-get update && apt-get install -y openssh-server sudo

RUN useradd -m -s /bin/bash ansible && \
    echo "ansible:ansible" | chpasswd && \
    usermod -aG sudo ansible && \
    echo "ansible ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

RUN mkdir -p /home/ansible/.ssh
COPY ansible_key.pub /home/ansible/.ssh/authorized_keys
RUN chown -R ansible:ansible /home/ansible/.ssh && \
    chmod 600 /home/ansible/.ssh/authorized_keys

RUN systemctl enable ssh

4. Dockerfile.rocky
Construye el nodo administrado Rocky Linux. Copia solo la clave pública

FROM geerlingguy/docker-rockylinux9-ansible:latest

RUN dnf -y install openssh-server sudo

RUN useradd -m -s /bin/bash ansible && \
    echo "ansible:ansible" | chpasswd && \
    usermod -aG wheel ansible && \
    echo "ansible ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

RUN mkdir -p /home/ansible/.ssh
COPY ansible_key.pub /home/ansible/.ssh/authorized_keys
RUN chown -R ansible:ansible /home/ansible/.ssh && \
    chmod 600 /home/ansible/.ssh/authorized_keys

RUN ssh-keygen -A
RUN systemctl enable sshd

4. Despliegue del Laboratorio
Una vez que tengas tu par de claves y los 4 archivos creados, levanta todo el entorno forzando la construcción de las imágenes:

Bash
docker compose up -d --build
Comprueba que los tres servicios estén corriendo sin errores:

Bash
docker compose ps
5. Verificación de Conectividad (¡Cero configuración manual!)
Como las imágenes ya instalaron todo y distribuyeron las claves SSH correctamente, puedes probar el entorno inmediatamente.

1. Ingresa al entorno web de VS Code:
Abre tu navegador web e ingresa a:

Bash
http://localhost:8443
Contraseña: ansible

2. Validar la instalación de Ansible:
Abre una terminal dentro de VS Code (Ctrl + ~). Notarás que ya eres el usuario ansible. Ejecuta:

Bash
ansible --version
3. Validar conectividad SSH automática:
Prueba conectarte a los nodos. Al tener las claves inyectadas por Docker, no te pedirá contraseña:

Bash
ssh ansible-ubuntu
(Escribe exit para regresar al nodo de control)

Bash
ssh ansible-rocky
(Escribe exit para regresar al nodo de control)

¡Tu laboratorio está listo! Ya puedes crear tu archivo de inventario y comenzar a escribir playbooks.


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

# Laboratorio 02: Configuración del Inventario y Archivo de Configuración de Ansible

En esta sección, configuraremos el inventario de servidores y el archivo principal de configuración de Ansible (`ansible.cfg`). Además, solucionaremos la advertencia de seguridad relacionada con los permisos de los directorios compartidos en Docker.

---

## Paso 1: Crear el archivo de inventario (`inventario.ini`)

En la terminal integrada de VS Code (dentro del directorio `~/workspace`), crea el archivo que definirá a qué contenedores nos vamos a conectar. Ejecuta el siguiente comando para generarlo:

```bash
cat << 'EOF' > inventario.ini
[web]
ansible-ubuntu

[db]
ansible-rocky

[all:vars]
ansible_user=ansible
EOF

Paso 2: Crear el archivo de configuración global (ansible.cfg)
Para no tener que especificar el inventario con el parámetro -i cada vez que ejecutemos un comando, crearemos el archivo ansible.cfg en el mismo directorio:

cat << 'EOF' > ansible.cfg
[defaults]
inventory = inventario.ini
remote_user = ansible
EOF

Paso 3: Forzar la lectura de configuración (ANSIBLE_CONFIG)
Dado que estamos trabajando dentro de un volumen compartido de Docker, Ansible detecta permisos muy abiertos (world-writable) y, por seguridad, ignora el archivo ansible.cfg.

Para indicarle explícitamente a Ansible que confíe en nuestro archivo, configuraremos la variable de entorno ANSIBLE_CONFIG. Además, la añadiremos al archivo ~/.bashrc para que sea permanente cada vez que abramos una nueva terminal. Ejecuta estos dos comandos:

# Aplicar para la sesión actual
export ANSIBLE_CONFIG="/home/ansible/workspace/ansible.cfg"

# Hacer la configuración permanente
echo 'export ANSIBLE_CONFIG="/home/ansible/workspace/ansible.cfg"' >> ~/.bashrc

Paso 4: Verificación del Inventario
Ahora podemos ejecutar comandos de Ansible directamente de forma más limpia. Ejecuta el siguiente comando para verificar que Ansible detecte correctamente tus nodos:

ansible all --list-hosts


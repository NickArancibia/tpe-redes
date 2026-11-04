# The Store sobre K3s gestionado con Ansible

TPE de Redes de Información (ITBA, 2C 2026) · Grupo 6 · Tema 5: Despliegue y gestión del clúster
de Kubernetes.

The Store es una tienda online hecha de cinco servicios (web, catálogo, carrito, pedidos y
checkout). Este repositorio arma la plataforma donde corre: con un comando se crean los
servidores, se instala Kubernetes (K3s) y se abre la tienda; con otro se suma o se retira un
servidor sin cerrarla; y si algo se rompe, se reconstruye todo igual que antes.

- [Cómo funciona](#cómo-funciona)
- [Antes de empezar](#antes-de-empezar)
- [Casos de uso](#casos-de-uso)
  1. [Armar todo desde cero](#1--armar-todo-desde-cero)
  2. [Abrir la tienda](#2--abrir-la-tienda)
  3. [Llega el Black Friday: sumar un servidor](#3--llega-el-black-friday-sumar-un-servidor)
  4. [Pasa el pico: retirar un servidor](#4--pasa-el-pico-retirar-un-servidor)
  5. [Se rompe todo: reconstruir](#5--se-rompe-todo-reconstruir)
- [Si algo no anda](#si-algo-no-anda)
- [Detalles técnicos](#detalles-técnicos)
- [Limitaciones](#limitaciones)

## Cómo funciona

Hay dos herramientas:

- **Ansible** lee una lista de servidores (`ansible/inventory/hosts.yml`) y ejecuta recetas
  (playbooks): una para armar los servidores, otra para abrir la tienda, otra para sumar un
  servidor, otra para retirarlo y otra para borrar todo. Antes de cambiar algo compara lo que hay
  con lo que pide la receta, así que correrla dos veces no rompe nada.
- **K3s** es un Kubernetes liviano. Reparte las partes de la tienda entre los servidores y la
  mantiene andando: si se retira un servidor, levanta su trabajo en los demás.

En esta prueba cada servidor es un contenedor Docker dentro de la misma computadora:

```
Tu computadora (Linux + Docker)
└── red k3s-net · 172.28.0.0/24
    ├── k3s-server   172.28.0.10   coordina el clúster (no corre partes de la tienda)
    ├── k3s-agent-1  172.28.0.11   corre partes de la tienda
    ├── k3s-agent-2  172.28.0.12   corre partes de la tienda
    └── k3s-agent-3  172.28.0.13   se suma en el caso 3
```

La tienda se abre en el navegador con la IP de cualquier servidor, por ejemplo
http://172.28.0.11.

| Receta (`ansible/`) | Caso | Qué hace |
|---|---|---|
| `create-cluster.yml` | 1 | Crea los servidores de la lista, instala K3s y los une |
| `deploy-store.yml` | 2, 3, 4 | Abre la tienda o cambia cuántas copias de la web corren |
| `add-nodes.yml` | 3 | Crea solo los servidores nuevos de la lista y los suma |
| `remove-node.yml` | 4 | Pasa el trabajo de un servidor a los demás y lo borra |
| `destroy.yml` | 5 | Borra servidores, red, datos y credenciales |

## Antes de empezar

### Qué hace falta

- **Linux con Docker Engine.** Probado en Ubuntu 24.04 con Docker 29. En Mac o Windows (Docker
  Desktop) no anda tal cual: ver [Si algo no anda](#si-algo-no-anda).
- **Ansible** (core 2.16 o más nuevo).
- **kubectl**, solo para mirar el clúster.
- Conexión a internet: los servidores descargan las imágenes de la tienda de GitHub Container
  Registry.

### Preparación (una sola vez)

Ansible necesita dos librerías de Python y dos colecciones de módulos:

```bash
# si Ansible se instaló con pipx:
pipx inject ansible-core docker kubernetes
# si se instaló con pip:
pip install docker kubernetes

cd ansible
ansible-galaxy collection install -r requirements.yml -p ./collections
```

**Todos los comandos de esta guía se corren desde `ansible/`.**

Las imágenes de la tienda ya están publicadas y son públicas
(`ghcr.io/nickarancibia/the-store-<servicio>:1.0.0`). No hace falta construir nada ni tener
credenciales de GitHub.

### Estado inicial

La lista de servidores arranca con un coordinador y dos workers. Si quedó `k3s-agent-3` de una
corrida anterior, sacalo de `inventory/hosts.yml` antes del caso 1:

```yaml
        k3s_agents:
          hosts:
            k3s-agent-1:
              node_ip: 172.28.0.11
            k3s-agent-2:
              node_ip: 172.28.0.12
```

## Casos de uso

Los cinco casos cuentan la vida de la tienda en orden: se arma, abre, crece para el Black
Friday, se achica cuando pasa el pico y se reconstruye después de una falla. Cada uno parte de
cómo terminó el anterior.

Para mirar el clúster con `kubectl`, después del caso 1:

```bash
export KUBECONFIG=$PWD/artifacts/kubeconfig
```

### 1 · Armar todo desde cero

Partimos de una computadora sin nada del proyecto y terminamos con tres servidores listos.

```bash
ansible-playbook create-cluster.yml
ansible-playbook create-cluster.yml     # segunda vez, a propósito
kubectl get nodes
```

**Qué tenés que ver**

- La primera vez, el resumen final (`PLAY RECAP`) muestra cambios (`changed`) en cada servidor.
- La segunda vez muestra `changed=0`: ya estaba todo, no hizo nada.
- `kubectl get nodes` lista `k3s-server`, `k3s-agent-1` y `k3s-agent-2` en estado `Ready`.

Tarda menos de un minuto.

### 2 · Abrir la tienda

```bash
ansible-playbook deploy-store.yml
kubectl get pods -n the-store -o wide
```

**Qué tenés que ver**

- Todos los pods en `Running`, repartidos entre `k3s-agent-1` y `k3s-agent-2`. Ninguno en
  `k3s-server`.
- Dos copias de la web (`ui-...`), una en cada worker.
- La tienda abre en http://172.28.0.11 y en http://172.28.0.12. Cualquier servidor atiende.

La primera vez cada servidor descarga las imágenes: puede tardar unos 5 minutos.

### 3 · Llega el Black Friday: sumar un servidor

Más clientes, así que pedimos una copia más de la web. Cada servidor acepta una sola copia, y
con dos workers la tercera no tiene dónde ir: queda esperando hasta que sumamos un servidor.

**1. Pedir la tercera copia:**

```bash
ansible-playbook deploy-store.yml -e store_ui_replicas=3
kubectl get pods -n the-store -l app.kubernetes.io/name=ui -o wide
```

La receta avisa `1 copia(s) de la web en Pending` y `kubectl` muestra esa copia en `Pending`.

**2. Agregar el servidor a la lista**, en `inventory/hosts.yml` dentro de `k3s_agents`:

```yaml
            k3s-agent-3:
              node_ip: 172.28.0.13
```

**3. Sumarlo:**

```bash
ansible-playbook add-nodes.yml
kubectl get nodes
kubectl get pods -n the-store -l app.kubernetes.io/name=ui -o wide
```

**Qué tenés que ver**

- En el resumen, solo `k3s-agent-3` tiene cambios; los demás quedan en `ok`.
- Cuatro servidores `Ready`.
- La copia que esperaba pasa a `Running` en `k3s-agent-3`. Antes puede pasar unos minutos en
  `ContainerCreating` mientras el servidor nuevo descarga la imagen.
- La tienda sigue abierta todo el tiempo.

El orden importa: la copia se pide **antes** de agregar el servidor a la lista, porque la receta
calcula cuántas copias entran según los workers que hay en la lista.

### 4 · Pasa el pico: retirar un servidor

Bajó el tráfico y no queremos seguir pagando un servidor de más.

**1. Volver a dos copias de la web:**

```bash
ansible-playbook deploy-store.yml
```

Sin `-e`, la receta usa el valor normal (2 copias). Si se saltea este paso, la copia del
servidor retirado se queda esperando lugar.

**2. Retirar un servidor:**

```bash
ansible-playbook remove-node.yml -e node=k3s-agent-1
```

La receta le avisa a Kubernetes que ese servidor no reciba más trabajo, mueve sus pods a los
demás, lo da de baja del clúster y borra el contenedor. Solo acepta workers: el coordinador no se
puede retirar.

**3. Sacarlo de la lista:** borrar `k3s-agent-1` de `inventory/hosts.yml`.

```bash
kubectl get nodes
kubectl get pods -n the-store -o wide
```

**Qué tenés que ver**

- `k3s-agent-1` ya no aparece en `kubectl get nodes` ni en `docker ps`.
- Sus pods corren ahora en `k3s-agent-2` y `k3s-agent-3`.
- La tienda sigue abriendo. Puede tardar unos segundos en responder mientras se mueven las partes
  que tienen una sola copia (ver [Limitaciones](#limitaciones)).

Se puede retirar cualquier worker. Retirar `k3s-agent-1` muestra mejor el movimiento de trabajo,
porque es el que tiene más pods.

### 5 · Se rompe todo: reconstruir

Supongamos lo peor: se perdió todo. Lo borramos y lo volvemos a levantar.

```bash
ansible-playbook destroy.yml
docker ps -a; docker network ls; docker volume ls     # no queda nada del proyecto
ansible-playbook create-cluster.yml
ansible-playbook deploy-store.yml
```

**Qué tenés que ver**

- Después de `destroy.yml`, ningún contenedor, red ni volumen del proyecto.
- Después de reconstruir, el clúster y la tienda quedan como al final del caso 2, con los
  servidores que tenga la lista en ese momento (después del caso 4: `k3s-server`, `k3s-agent-2`
  y `k3s-agent-3`).

Borrar también elimina las imágenes descargadas, así que abrir la tienda vuelve a tardar unos 5
minutos. `destroy.yml` borra incluso servidores que ya no están en la lista, porque los busca por
una etiqueta del proyecto.

## Si algo no anda

| Síntoma | Causa probable | Qué hacer |
|---|---|---|
| `permission denied` sobre `/var/run/docker.sock` | Tu usuario no puede usar Docker | Agregarlo al grupo `docker` y volver a iniciar sesión |
| `Failed to import the required Python library (docker)` o `(kubernetes)` | Faltan las librerías en el Python de Ansible | Repetir la [preparación](#preparación-una-sola-vez) |
| `couldn't resolve module/action 'community.docker...'` | Faltan las colecciones | `ansible-galaxy collection install -r requirements.yml -p ./collections` desde `ansible/` |
| `kubectl` no conecta | Falta el kubeconfig | `export KUBECONFIG=$PWD/artifacts/kubeconfig` desde `ansible/` |
| No se llega a `172.28.0.x` (Mac o Windows) | Docker Desktop corre los contenedores en una VM y la computadora no tiene ruta a esa red | Usar Linux con Docker Engine |
| La red `k3s-net` no se crea | Otra red Docker ya usa `172.28.0.0/24` | Borrar esa red o cambiar la subred en `inventory/group_vars/all.yml` |
| Pods en `ContainerCreating` varios minutos | Están descargando las imágenes | Esperar; con mala conexión tarda más |

## Detalles técnicos

Esta sección explica cómo están armados los archivos. No hace falta leerla para correr los casos.

### Estructura

```
ansible/
├── ansible.cfg                   inventario, rutas de roles y colecciones
├── requirements.yml              colecciones community.docker y kubernetes.core
├── inventory/
│   ├── hosts.yml                 qué servidores existen: nombre, grupo (rol) e IP
│   └── group_vars/
│       ├── all.yml               red, versión de K3s, registry, copias de la web
│       ├── k3s_server.yml        con qué comando arranca K3s en el coordinador
│       └── k3s_agents.yml        con qué comando arranca K3s en los workers
├── roles/
│   ├── docker_node/              crea (main.yml) o borra (absent.yml) el contenedor-servidor
│   ├── k3s/                      espera la API, guarda el kubeconfig y espera Ready
│   └── the_store/                aplica los manifiestos y espera los servicios
├── create-cluster.yml            caso 1
├── deploy-store.yml              casos 2, 3 y 4
├── add-nodes.yml                 caso 3
├── remove-node.yml               caso 4
├── destroy.yml                   caso 5
└── artifacts/                    se genera (no se versiona): kubeconfig y token
the-store-main/                   código original de The Store
```

La creación del servidor (rol `docker_node`) está separada de la instalación de K3s (rol `k3s` y
`group_vars`). Para usar máquinas virtuales en lugar de contenedores alcanza con reemplazar ese
rol.

### Cómo se conecta Ansible

Ansible no entra a los servidores por SSH (`ansible_connection: local`). Todo corre en tu
computadora y usa dos APIs:

| Para qué | Cómo llega | Credencial |
|---|---|---|
| Crear y borrar servidores, red y volúmenes | Socket de Docker (`/var/run/docker.sock`) | Permiso sobre el socket |
| Desplegar, drenar y verificar | API de Kubernetes, `https://172.28.0.10:6443` | `artifacts/kubeconfig` |

La única vez que Ansible ejecuta algo adentro de un servidor es para leer el kubeconfig que
genera K3s, y lo hace con `docker exec`.

### Red

| Red | Rango | Uso |
|---|---|---|
| `k3s-net` (Docker, bridge) | 172.28.0.0/24, gateway .1 | IP fija de cada servidor, según el inventario |
| Pods | 10.42.0.0/16 (un /24 por servidor) | Flannel; entre servidores viaja en VXLAN (UDP 8472) |
| Services | 10.43.0.0/16 | IPs internas de cada servicio; CoreDNS las resuelve por nombre |

En Linux, Docker crea una interfaz en la computadora para `k3s-net`, así que la computadora llega
directo a `172.28.0.x` sin publicar puertos.

La entrada de los clientes es Traefik con ServiceLB, que vienen con K3s y publican el puerto 80
en cada servidor. El Ingress manda todo el tráfico a la web.

### Servidores y clúster

- **Contenedores:** imagen `rancher/k3s:v1.33.4-k3s1`, en modo privilegiado (K3s administra
  iptables y cgroups de los pods), con `tmpfs` en `/run` y `/var/run` y un volumen
  `<servidor>-data` en `/var/lib/rancher/k3s`.
- **Arranque:** la imagen ejecuta el binario `k3s`; Ansible le pasa `server ...` o `agent ...`
  según el grupo del servidor en el inventario.
- **Unión al clúster:** los workers arrancan con `K3S_URL=https://172.28.0.10:6443` y un token
  compartido. Ansible genera el token una sola vez en `artifacts/k3s-token`.
- **Coordinador dedicado:** el server tiene el taint
  `node-role.kubernetes.io/control-plane:NoSchedule`, así las partes de la tienda corren solo en
  los workers.
- **Kubeconfig:** se copia del server y se reescribe para apuntar a `https://172.28.0.10:6443`.
  Esa IP está en el certificado gracias a `--tls-san`.
- **Etiqueta del proyecto:** red, volúmenes y contenedores llevan
  `com.the-store.project=the-store-k3s`. `destroy.yml` borra todo lo que tenga esa etiqueta.

### La tienda

- **Manifiestos:** `roles/the_store/templates/the-store.yaml.j2` es una copia de
  `the-store-main/dist/kubernetes.yaml` con tres cambios: las imágenes vienen de
  `ghcr.io/nickarancibia`, el Ingress usa Traefik y la web tiene copias configurables con
  anti-afinidad.
- **Copias de la web:** `store_ui_replicas` en `inventory/group_vars/all.yml` (2 por defecto).
  La web tiene anti-afinidad obligatoria por servidor (`topologyKey: kubernetes.io/hostname`):
  nunca hay dos copias en el mismo worker. Si se piden más copias que workers, las que sobran
  quedan en `Pending`. `deploy-store.yml` espera solo las copias que entran y avisa cuántas
  quedaron esperando.
- **Imágenes:** cada servidor las descarga por su cuenta, así un servidor nuevo no depende de que
  le copien nada.

### Variables principales (`inventory/group_vars/all.yml`)

| Variable | Valor | Para qué |
|---|---|---|
| `k3s_network_subnet` | `172.28.0.0/24` | Subred de `k3s-net` |
| `k3s_image` | `rancher/k3s:v1.33.4-k3s1` | Versión de K3s |
| `store_registry` | `ghcr.io/nickarancibia` | De dónde se bajan las imágenes |
| `store_image_tag` | `1.0.0` | Versión de las imágenes |
| `store_ui_replicas` | `2` | Copias de la web; se cambia en una corrida con `-e store_ui_replicas=3` |

## Limitaciones

- Catálogo, carrito, pedidos, checkout y Traefik tienen una sola copia. Al retirar el servidor que
  los tiene, la tienda puede no responder unos segundos mientras se levantan en otro.
- Hay un solo coordinador: si se cae, el clúster no acepta cambios hasta que vuelva.
- Los datos de la tienda (carritos, pedidos) se guardan en memoria y se pierden al reconstruir.
- Kubernetes prefiere los servidores que ya tienen las imágenes, así que un servidor recién
  sumado puede no recibir partes de la tienda al principio. La copia extra de la web sí va ahí,
  porque es el único lugar que la acepta.
- Todo corre en una sola computadora: los servidores y la tienda solo son accesibles desde ella.

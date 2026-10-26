# The Store sobre K3s gestionado con Ansible

TPE de Redes de Información (ITBA, 2C 2026) · Grupo 6 · Tema 5: Despliegue y gestión del clúster de Kubernetes.

Ansible crea los nodos, instala el clúster K3s y despliega The Store. Cada nodo es un
contenedor Docker basado en la imagen oficial `rancher/k3s`, conectado a la red `k3s-net`
con la IP fija que indica el inventario.

```
Host (Ubuntu 24.04 + Docker Engine)
└── red Docker k3s-net · 172.28.0.0/24
    ├── k3s-server   172.28.0.10   control plane (API :6443)
    ├── k3s-agent-1  172.28.0.11   worker
    ├── k3s-agent-2  172.28.0.12   worker
    └── k3s-agent-3  172.28.0.13   worker que se agrega en el caso 3
Pods: 10.42.0.0/16 (un /24 por nodo) · Services: 10.43.0.0/16
Imágenes de la tienda: ghcr.io/nickarancibia/the-store-<servicio>:1.0.0
Entrada: Traefik + ServiceLB, HTTP :80 en la IP de cualquier nodo
```

## Estructura

```
ansible/
├── ansible.cfg
├── requirements.yml              colecciones community.docker y kubernetes.core
├── inventory/
│   ├── hosts.yml                 qué nodos existen: nombre, rol (grupo) e IP
│   └── group_vars/
│       ├── all.yml               red, versión de K3s, registry, rutas
│       ├── k3s_server.yml        cómo arranca K3s en el control plane
│       └── k3s_agents.yml        cómo arranca K3s en los workers
├── roles/
│   ├── docker_node/              crea/elimina el nodo (contenedor, volumen, red)
│   ├── k3s/                      espera la API, guarda el kubeconfig, espera Ready
│   └── the_store/                aplica los manifiestos y espera los servicios
├── create-cluster.yml            caso 1
├── deploy-store.yml              caso 2
├── add-nodes.yml                 caso 3
├── remove-node.yml               caso 4
├── destroy.yml                   caso 5
├── publish-images.yml            construye y sube las imágenes a GHCR
└── artifacts/                    (generado, no se versiona) kubeconfig y token
the-store-main/                   código de The Store
```

La creación de la "máquina" (rol `docker_node`) está separada de la configuración de K3s
(rol `k3s` + `group_vars`), así que el proveedor de nodos se podría cambiar por VMs sin tocar
el resto.

## Requisitos

- Linux con Docker Engine (probado en Ubuntu 24.04, Docker 29, cgroups v2).
- Ansible core ≥ 2.16, `kubectl` (solo para mirar el clúster).
- Librerías Python `docker` y `kubernetes` en el Python de Ansible.

```bash
# si Ansible se instaló con pipx:
pipx inject ansible-core docker kubernetes
# si se instaló con pip:  pip install docker kubernetes

cd ansible
ansible-galaxy collection install -r requirements.yml -p ./collections
```

Todos los comandos siguientes se corren desde `ansible/`.

## 0 · Publicar las imágenes (una sola vez)

Los nodos descargan las imágenes de GitHub Container Registry. Para subirlas hace falta un
token de GitHub (classic) con el permiso `write:packages`:

```bash
echo "$GHCR_TOKEN" | docker login ghcr.io -u <usuario> --password-stdin
ansible-playbook publish-images.yml
```

Después, en GitHub → Packages, poner cada paquete `the-store-*` como **Public** para que los
nodos puedan descargarlo sin credenciales. El owner y el tag se cambian en
`inventory/group_vars/all.yml` (`store_registry`, `store_image_tag`).

## Casos de uso

```bash
export KUBECONFIG=$PWD/artifacts/kubeconfig   # lo genera create-cluster.yml
```

### 1 · Reproducibilidad: levantar el clúster desde cero

```bash
ansible-playbook create-cluster.yml   # crea red, volúmenes y 3 nodos; espera que estén Ready
ansible-playbook create-cluster.yml   # segunda vez: changed=0 en todos los nodos
docker ps
kubectl get nodes -o wide
```

### 2 · Validación: desplegar The Store

```bash
ansible-playbook deploy-store.yml
kubectl get pods -n the-store -o wide
```

La tienda abre en http://172.28.0.10, http://172.28.0.11 o http://172.28.0.12.

### 3 · Capacidad: sumar un nodo

Agregar el worker en `inventory/hosts.yml`, dentro de `k3s_agents`:

```yaml
            k3s-agent-3:
              node_ip: 172.28.0.13
```

```bash
ansible-playbook add-nodes.yml        # solo k3s-agent-3 tiene cambios
docker ps
kubectl get nodes                     # 4 nodos Ready
```

### 4 · Mantenimiento: retirar un nodo

```bash
ansible-playbook remove-node.yml -e node=k3s-agent-1
```

El playbook hace cordon y drain del nodo (Kubernetes recrea sus pods en los demás nodos),
lo borra del clúster y elimina su contenedor y su volumen. Después hay que sacar el nodo de
`inventory/hosts.yml`. Solo acepta workers: el server no se puede retirar.

```bash
kubectl get nodes
kubectl get pods -n the-store -o wide
docker ps
```

### 5 · Reconstrucción: destruir y volver a crear

```bash
ansible-playbook destroy.yml
docker ps -a; docker network ls; docker volume ls   # nada del proyecto
ansible-playbook create-cluster.yml
ansible-playbook deploy-store.yml
```

`destroy.yml` busca los recursos por la etiqueta `com.the-store.project=the-store-k3s`, así
también borra nodos que ya no figuran en el inventario. Además elimina `artifacts/`
(kubeconfig y token).

## Detalles técnicos

- **Nodos:** contenedores `rancher/k3s:v1.33.4-k3s1` privilegiados (K3s administra iptables y
  cgroups de los pods), con `tmpfs` en `/run` y `/var/run`, y un volumen `<nodo>-data` en
  `/var/lib/rancher/k3s` (estado de K3s e imágenes de containerd).
- **Unión al clúster:** los agents arrancan con `K3S_URL=https://172.28.0.10:6443` y el token
  compartido, que Ansible genera una sola vez en `artifacts/k3s-token`.
- **Control plane dedicado:** el server tiene el taint
  `node-role.kubernetes.io/control-plane:NoSchedule`, así los pods de The Store corren en los
  workers.
- **Acceso:** el kubeconfig se copia del server y se reescribe para apuntar a
  `https://172.28.0.10:6443` (la IP está en los SAN del certificado con `--tls-san`).
- **Manifiestos:** `roles/the_store/templates/the-store.yaml.j2` es
  `the-store-main/dist/kubernetes.yaml` con las imágenes apuntando al registry y el Ingress
  con `ingressClassName: traefik` sin host fijo.
- **Idempotencia:** los módulos de `community.docker` y `kubernetes.core` comparan el estado
  actual con el deseado y solo cambian lo necesario.

## Limitaciones conocidas

- Cada servicio de The Store y Traefik tienen una sola réplica. Si se drena el nodo que los
  tiene, la tienda deja de responder unos segundos hasta que los pods se recrean en otro nodo.
- Por la localidad de imágenes, el scheduler suele preferir los nodos que ya las tienen
  descargadas, así que un nodo recién agregado puede no recibir pods de la tienda al principio.

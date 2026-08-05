# Desplegar un clúster k8s con Ansible

Mantenedores: [zouox](https://gitlab.com/zouox), [ipefix ledruide](https://gitlab.com/ipefixledruide)

Distribuciones compatibles:

- Debian 12
- Familia RHEL 8+ (Almalinux, Rocky Linux, ...)

Arquitectura compatible:

- x64

Este rol automatiza la configuración de Kubernetes con la integración de componentes esenciales como un controlador de entrada (Traefik o Nginx), una clase de almacenamiento (local o NFS) y una IP Virtual (kube-vip) para alta disponibilidad y equilibrio de carga.

⚠️ Todos los problemas deben crearse en el [repositorio principal del proyecto](https://gitlab.com/zouox-projects/ansible/zouox.ansible_k8s). \
⚠️ El [repositorio de Github](https://github.com/zouox/zouox.ansible_k8s) es un espejo para Ansible Galaxy. Puede visitar el repositorio oficial en [Gitlab](https://gitlab.com/zouox-projects/ansible/zouox.ansible_k8s)

## Requisitos

- Acceso root (o un usuario con permisos sudo) en todos los nodos gestionados del inventario

- La colección de módulos kubernetes.core debe estar instalada en el controlador de Ansible:

```bash
ansible-galaxy collection install kubernetes.core
```

- La colección de módulos community.general debe estar instalada en el controlador de Ansible:

```bash
ansible-galaxy collection install community.general
```

## Uso

### Preparación

Cree un inventario para el clúster. **No cambie los nombres de los grupos ya que este rol utiliza nombres de grupos fijos** (por ahora, en cuanto encontremos una solución adecuada)

```yaml
all:
  children:
    k8s_main_master:
      hosts:
        k8s-cp-1:
          ansible_host: 192.168.10.51
    k8s_other_masters:
      hosts:
        k8s-cp-2:
          ansible_host: 192.168.10.52
        k8s-cp-3:
          ansible_host: 192.168.10.53
    k8s_workers:
      hosts:
        k8s-worker-1:
          ansible_host: 192.168.10.61
```

### Variables

|Nombre |Descripción  |Predeterminado  |Requerido |
|---|---|---|---|
|set_hostnames  |Establece automáticamente los nombres de host para los hosts de destino, basándose en el nombre del inventario de Ansible. |True |No  |
|kubernetes_version  |Establece la versión de Kubernetes. |v1.30 |No  |
|pod_network_cidr  |Establece el CIDR de red de pods para el clúster|10.244.0.0/16 |No  |
|kubernetes_cni  |Establece la CNI instalada. Las opciones son: flannel, calico, false |flannel |No  |
|calico_version  |Establece la versión de Calico a descargar. |v3.28.0 |No |
|controlplane_endpoint  |Establece el punto final del plano de control. Puede ser un dominio o una dirección IP. |ansible_hostname |No  |
|kubevip_install |Instala kube-vip en el clúster. Implica la instalación de un clúster en alta disponibilidad. Establezca en **False** para una instalación de nodo único.  |True |No |
|kubernetes_ingress |Establece el controlador de entrada instalado. Las opciones son: traefik, nginx, false |traefik |No  |
|kubernetes_storageclass |Establece la clase de almacenamiento instalada. Las opciones son: local, nfs, false |local |No  |
|nfs_server_address  |La dirección IP del servidor NFS para la clase de almacenamiento NFS. |None |Sí, si la variable *kubernetes_storageclass* se establece en **nfs** |
|nfs_server_path  |La ruta en el servidor NFS para la clase de almacenamiento NFS. |None |Sí, si la variable *kubernetes_storageclass* se establece en **nfs** |
|vip_address  |La dirección IP que kube-vip anunciará. |None |Sí, si la variable *kubevip_install* se establece en **true** |
|vip_interface |La interfaz que kube-vip utilizará para anunciar la IP virtual. |ansible_facts.default_ipv4.interface  |No |

### Etiquetas de Ansible

|Nombre |Descripción  |
|---|---|
|install  |Ejecuta todas las tareas. Use esta etiqueta para una instalación completa |
|prepare  |Ejecuta tareas relacionadas con la preparación de los nodos. |
|k8s  |Ejecuta todas las tareas relacionadas con k8s (como unir miembros al clúster)  |
|main_master  |Ejecuta todas las tareas de k8s relativas a la instalación del primer nodo del plano de control  |
|other_master  |Ejecuta todas las tareas de k8s relativas a la instalación de los nodos del plano de control adicionales |
|worker |Ejecuta todas las tareas de k8s relativas a la instalación de los nodos trabajadores |

### Instalación

```bash
# Por su propia seguridad
export ANSIBLE_HOST_KEY_CHECKING=False
```

#### En modo de alta disponibilidad

##### Instalación completa

```yaml
ansible-playbook -i /path/to/inventory --user <ansible_user> --private-key </path/to/ansible_user_private_key> -t install <playbook_name>.yml
```

##### Unir nuevos miembros al clúster

```yaml
# Añadir nuevos planos de control
ansible-playbook -i /path/to/inventory --user <ansible_user> --private-key </path/to/ansible_user_private_key> -t other_master <playbook_name>.yml
```

```yaml
# Añadir nuevos nodos trabajadores
ansible-playbook -i /path/to/inventory --user <ansible_user> --private-key </path/to/ansible_user_private_key> -t worker <playbook_name>.yml
```

#### En modo de nodo único

```yaml
ansible-playbook -i /path/to/inventory --user <ansible_user> --private-key </path/to/ansible_user_private_key> -t install --skip-tags worker, other_master <playbook_name>.yml
```

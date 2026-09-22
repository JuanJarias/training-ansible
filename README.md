# Provisionamiento de una VM Azure con Ansible y Docker

Este repositorio contiene la automatización con Ansible para configurar una máquina virtual Ubuntu creada previamente en Microsoft Azure con Terraform. La práctica instala Docker y ejecuta un contenedor de Super Mario Bros usando la imagen `pengbai/docker-supermario:latest`.

## Objetivo

Provisionar una VM Azure ya existente mediante Ansible, separando las responsabilidades de infraestructura y configuración:

| Herramienta | Responsabilidad |
| --- | --- |
| Terraform | Crea y administra la infraestructura de Azure: VM, red, IP pública, NSG e interfaz de red. |
| Ansible | Se conecta por SSH a la VM y automatiza la instalación de Docker y la ejecución del contenedor. |
| Docker | Ejecuta la aplicación web Super Mario Bros de forma aislada en un contenedor. |

## Arquitectura

```text
Equipo Fedora (nodo de control)
        |
        | SSH con clave privada id_ed25519
        v
VM Ubuntu en Microsoft Azure
        |
        | Ansible
        v
Docker Engine
        |
        | Puerto host 8787 -> puerto contenedor 8080
        v
Contenedor Super Mario Bros
```

El contenedor se publica con el mapeo `8787:8080`. Por ello, una vez habilitada la regla de red correspondiente en Azure, el juego se abre en:

```text
http://IP_PUBLICA_DE_LA_VM:8787
```

## Estructura del repositorio

```text
training-ansible/
├── ansible.cfg
├── inventory/
│   └── hosts.ini
├── playbooks/
│   ├── install_docker.yml
│   └── run_container.yml
└── roles/
    └── docker_container/
```

- `ansible.cfg`: configuración local del proyecto Ansible.
- `inventory/hosts.ini`: inventario con la VM Azure administrada.
- `playbooks/install_docker.yml`: playbook para instalar y preparar Docker en la VM.
- `playbooks/run_container.yml`: playbook que ejecuta el contenedor de Super Mario Bros.
- `roles/docker_container/`: rol utilizado para gestionar la ejecución del contenedor.

## Prerrequisitos

Antes de ejecutar los playbooks se requiere:

- Una VM Ubuntu creada y encendida en Azure.
- Una dirección IP pública asignada a la VM.
- Puerto TCP 22 permitido en el NSG de Azure desde la IP pública actual del equipo de control.
- Ansible instalado en Fedora/Linux.
- Una clave privada SSH que corresponda con la clave pública configurada en la VM.
- Docker instalado mediante el playbook antes de ejecutar el contenedor.
- Puerto TCP 8787 permitido en el NSG de Azure para acceder al juego desde un navegador.

Instalación de Ansible en Fedora:

```bash
sudo dnf install ansible -y
ansible --version
```

## Configuración del inventario

El inventario se encuentra en `inventory/hosts.ini`. Debe apuntar a la IP pública actual de la VM y a la clave privada SSH local.

```ini
[azure_vm]
vm_azure ansible_host=IP_PUBLICA_DE_LA_VM

[azure_vm:vars]
ansible_user=azureuser
ansible_ssh_private_key_file=/home/USUARIO/.ssh/id_ed25519
ansible_python_interpreter=/usr/bin/python3
```

Reemplace los siguientes valores según su entorno:

- `IP_PUBLICA_DE_LA_VM`: IP pública generada por Terraform o visible en Azure Portal.
- `/home/USUARIO/.ssh/id_ed25519`: ruta absoluta de la clave **privada** SSH.
- `azureuser`: usuario administrador definido al crear la VM.

> Nunca agregue al repositorio la clave privada `id_ed25519`, credenciales de Azure ni archivos que contengan secretos.

Asegure permisos adecuados para la clave privada:

```bash
chmod 600 ~/.ssh/id_ed25519
```

## Verificar conectividad

Antes de ejecutar un playbook, compruebe que SSH funciona de manera directa:

```bash
ssh -i ~/.ssh/id_ed25519 azureuser@IP_PUBLICA_DE_LA_VM
```

Después pruebe la conectividad administrada por Ansible:

```bash
ansible -i inventory/hosts.ini azure_vm -m ping
```

Resultado esperado:

```text
vm_azure | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

## Ejecución

### 1. Instalar Docker

Ejecute el playbook de instalación:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/install_docker.yml
```

Este paso prepara la VM para administrar contenedores Docker.

### 2. Ejecutar Super Mario Bros

Ejecute el playbook del contenedor:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/run_container.yml
```

El playbook crea o garantiza la ejecución del contenedor `supermario-container` a partir de la imagen:

```text
pengbai/docker-supermario:latest
```

Un resumen exitoso debe terminar con valores similares a estos:

```text
PLAY RECAP
vm_azure : ok=2 changed=1 unreachable=0 failed=0
```

### 3. Verificar el contenedor

Consulte el estado remoto de Docker mediante Ansible:

```bash
ansible -i inventory/hosts.ini azure_vm -a "sudo docker ps"
```

La salida esperada incluye el mapeo de puertos:

```text
0.0.0.0:8787->8080/tcp
```

Esto indica que el puerto `8787` de la VM redirige al puerto `8080` dentro del contenedor.

## Regla de red Azure

El NSG de Azure debe permitir tráfico de entrada TCP al puerto `8787` para acceder públicamente al juego. Si la infraestructura se administra con Terraform, agregue una regla similar al recurso `azurerm_network_security_group`:

```hcl
security_rule {
  name                       = "Allow-Mario-Web"
  priority                   = 1002
  direction                  = "Inbound"
  access                     = "Allow"
  protocol                   = "Tcp"
  source_port_range          = "*"
  destination_port_range     = "8787"
  source_address_prefix      = "*"
  destination_address_prefix = "*"
}
```

Después aplique el cambio desde el proyecto Terraform:

```bash
terraform fmt
terraform plan
terraform apply
```

> Para una práctica académica la regla pública facilita la demostración. En un entorno real se recomienda restringir `source_address_prefix` a redes o IPs autorizadas.

## Acceder al juego

Cuando Docker esté activo y el NSG permita el puerto `8787`, abra en un navegador:

```text
http://IP_PUBLICA_DE_LA_VM:8787
```

Ejemplo:

```text
http://20.127.73.126:8787
```

## Idempotencia

Ansible trabaja de forma declarativa: los playbooks describen el estado esperado. Al ejecutarlos nuevamente, Ansible debe mantener Docker y el contenedor en el estado requerido sin recrear innecesariamente recursos que ya cumplen la configuración.

Puede comprobarlo ejecutando otra vez:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/run_container.yml
```

Si la configuración no necesita cambios, el resumen debe mostrar `changed=0` o un número menor de cambios que en la primera ejecución.

## Solución de problemas

### Inventario vacío o grupo no encontrado

Si aparece un aviso como `provided hosts list is empty` o `Could not match supplied host pattern`, especifique el inventario:

```bash
ansible -i inventory/hosts.ini azure_vm -m ping
```

### Host inaccesible

Si aparece `UNREACHABLE`, revise:

- Que la VM esté en estado **Running** en Azure.
- Que la IP pública de `hosts.ini` sea correcta.
- Que el NSG permita TCP/22 desde su IP actual.
- Que `ansible_ssh_private_key_file` apunte a la clave privada correcta.
- Que el usuario `azureuser` coincida con el definido en Terraform.

### El contenedor está activo pero el juego no carga

Ejecute:

```bash
ansible -i inventory/hosts.ini azure_vm -a "sudo docker ps"
```

Compruebe el mapeo `0.0.0.0:8787->8080/tcp` y confirme que el NSG de Azure permite TCP/8787.

### La IP pública cambia

Si se modifica la IP de la VM, actualice `ansible_host` en `inventory/hosts.ini`. Si cambia la IP pública desde la que administra la máquina, actualice la regla SSH del NSG en el proyecto Terraform y ejecute `terraform apply`.

## Evidencias recomendadas

Para documentar la práctica, agregue capturas de pantalla de:

1. `ansible -i inventory/hosts.ini azure_vm -m ping` con resultado `pong`.
2. Ejecución exitosa de `install_docker.yml`.
3. Ejecución exitosa de `run_container.yml`.
4. Resultado de `sudo docker ps` mostrando `0.0.0.0:8787->8080/tcp`.
5. Juego Super Mario Bros abierto desde el navegador mediante la IP pública de Azure.

## Autor

Juan José Arias Gallego  
Estudiante — Universidad ICESI

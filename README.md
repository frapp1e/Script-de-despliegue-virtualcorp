VirtualCorp Lab Deployment Script

Un script en Bash que automatiza la creación de un laboratorio completo en VirtualBox. Permite desplegar routers, servidores y PCs clientes distribuidos en DMZ, redes privadas y zonas de desarrollo.

Funcionalidades:

Importación de máquinas OVA como plantillas base.

Clonado y configuración de routers, servidores y clientes.

Configuración de interfaces de red según subredes definidas (DMZ, privada, pisos de oficina).

Gestión de impresoras virtuales y máquinas de desarrollo (www/data servers).

Snapshots automáticos de cada máquina para restauración rápida.

Listado, eliminación y reinicio de grupos de máquinas mediante parámetros.

Compatible con Linux nativo y entornos WSL.

Uso:

./deploy_lab.sh -i           # Importar OVA
./deploy_lab.sh -r           # Crear routers
./deploy_lab.sh -f 3         # Crear clientes en 3 pisos
./deploy_lab.sh -s           # Crear servidores DMZ y privados
./deploy_lab.sh --list       # Listar todas las VMs

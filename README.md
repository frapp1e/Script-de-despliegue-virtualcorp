# Script-de-despliegue-virtualcorp
#!/bin/bash

# -------------------------------
# Configuración inicial
# -------------------------------
ovaFile=VirtualLab.ova
baseGroupName="/VCorp"
router_template=openwrt_template
server_template=debian_server_template
basic_client_template=debian_desktop_template

# Detectar entorno: WSL o Linux nativo
if grep -q microsoft /proc/version; then
    echo "Configurando para entorno WSL"
    vbox="/mnt/c/Program Files/Oracle/VirtualBox/VBoxManage.exe"
    default_vm_location="/mnt/c/VirtualCorp"
    wsl_default_vm_location="C:\\VirtualCorp"
else
    echo "Configurando para entorno Linux nativo"
    vbox="vboxmanage"
    default_vm_location="$HOME/VirtualBox VMs"
    wsl_default_vm_location="$HOME/VirtualBox VMs"
fi

# PCs de planta 0
pcs_pl0=( "expo1" "expo2" "expo3" "expo4" "conserge1" "conserge2" )

# Redes
declare -A networks=(
    ["VCorp_DMZ"]="10.1.0.192/27"
    ["VCorp_private"]="10.1.0.128/26"
    ["VCorp_pl0"]="10.1.0.0/25"
    ["VCorp_pl1"]="10.1.1.0/24"
    ["VCorp_pl2"]="10.1.2.0/24"
    ["VCorp_pl3"]="10.1.3.0/25"
    ["VCorp_direccion"]="10.1.3.128/25"
    ["VCorp_rt0_1"]="10.1.0.224/30"
    ["VCorp_rt1_2"]="10.1.0.228/30"
    ["VCorp_rt2_3"]="10.1.0.232/30"
)

# Impresoras por planta
printers=(
    "planta01 printer_pl01 VCorp_pl1"
    "planta02 printer_pl02 VCorp_pl2"
    "planta03 printer_pl03 VCorp_pl3"
    "plantadireccion printer_direccion VCorp_direccion"
)

# -------------------------------
# Funciones principales
# -------------------------------

get_all_vms_uuid() {
    "$vbox" list vms | sed 's/.*{\(.*\)}/\1/'
}

get_vms_group() {
    "$vbox" list vms -l | grep -E "^Groups:|^Name:" | sed -E "s/.*:\s+//" | while read -r vmname; do
        read -r vmgroup
        echo "$vmname $vmgroup"
    done
}

get_vms_of_group() {
    get_all_vms_uuid | while read -r uuid; do
        uuid=$(echo "$uuid" | sed 's/\r//g') # eliminar CR de Windows
        "$vbox" showvminfo "$uuid" | grep -E "^Groups:\s+$1" >/dev/null && echo "$uuid"
    done
}

rm_vms_of_group() {
    get_vms_of_group "$1" | while read -r uuid; do
        echo "$uuid"
        "$vbox" unregistervm "$uuid" --delete-all >/dev/null
    done
}

list_vms_names() {
    get_vms_group | while read -r vmname vmgroup; do
        if [[ $vmgroup =~ ^${baseGroupName}[a-zA-Z0-9_/]*$ ]]; then
            echo "$vmname"
        fi
    done
}

# Crear impresoras
create_printers() {
    for printer in "${printers[@]}"; do
        echo "$printer" | { read -r pr_group pr_name pr_net
            "$vbox" clonevm "$basic_client_template" \
                --groups "$baseGroupName/$pr_group" \
                --name "$pr_name" --register --snapshot base --options=Link
            "$vbox" modifyvm "$pr_name" --groups "$baseGroupName/$pr_group" --nic1 intnet --intnet1 "$pr_net"
        }
    done
}

rm_printers() {
    for printer in "${printers[@]}"; do
        echo "$printer" | { read -r pr_group pr_name pr_net
            "$vbox" unregistervm "$pr_name" --delete-all
        }
    done
}

# Importar OVA
import_ova() {
    "$vbox" import "$ovaFile" \
        --vsys=0 --vmname "${server_template}" --group "$baseGroupName/templates" \
        --vsys=1 --vmname "${router_template}" --group "$baseGroupName/templates" \
        --vsys=2 --vmname "${basic_client_template}" --group "$baseGroupName/templates"
    
    echo "VMs imported. Taking snapshots..."
    "$vbox" snapshot "${router_template}" take "base"
    "$vbox" snapshot "${server_template}" take "base"
    "$vbox" snapshot "${basic_client_template}" take "base"
    echo "Snapshots done."
}

# Crear routers
create_routers() {
    for i in {0..3}; do
        "$vbox" clonevm "$router_template" \
            --groups "$baseGroupName/routers" \
            --name router$i --register --snapshot base
        "$vbox" modifyvm router$i --groups "$baseGroupName/routers"
    done

    # Configuración interfaces
    "$vbox" modifyvm router0 --nic1 nat --nic2 intnet --intnet2 VCorp_DMZ --nic3 intnet --intnet3 VCorp_rt0_1
    "$vbox" modifyvm router1 --nic1 intnet --intnet1 VCorp_rt0_1 --nic2 intnet --intnet2 VCorp_private --nic3 intnet --intnet3 VCorp_rt1_2
    "$vbox" modifyvm router2 --nic1 intnet --intnet1 VCorp_rt1_2 --nic2 intnet --intnet2 VCorp_pl0 --nic3 intnet --intnet3 VCorp_pl1 --nic4 intnet --intnet4 VCorp_rt2_3
    "$vbox" modifyvm router3 --nic1 intnet --intnet1 VCorp_rt2_3 --nic2 intnet --intnet2 VCorp_pl2 --nic3 intnet --intnet3 VCorp_pl3 --nic4 intnet --intnet4 VCorp_direccion
}

# Crear servidores DMZ
create_dmz() {
    echo "$1" | { IFS=, read -ra srv_list
        for name in "${srv_list[@]}"; do
            "$vbox" clonevm "$server_template" \
                --groups "$baseGroupName/dmz" --name "$name" --register --snapshot base --options=Link
            "$vbox" modifyvm "$name" --groups "$baseGroupName/dmz" --nic1 intnet --intnet1 VCorp_DMZ
        done
    }
}

# Crear servidores privados
create_private() {
    for name in intrasrv filesrv datasrv ldapsrv; do
        "$vbox" clonevm "$server_template" \
            --groups "$baseGroupName/private" --name "$name" --register --snapshot base --options=Link
        "$vbox" modifyvm "$name" --groups "$baseGroupName/private" --nic1 intnet --intnet1 VCorp_private
    done
}

# Crear PCs por planta
create_floor() {
    local floor="$1"
    local num_clients="$2"
    local nat_network="$3"

    for iClient in $(seq -w "$num_clients"); do
        "$vbox" clonevm "$basic_client_template" \
            --name "pc${iClient}_pl${floor}" --register --snapshot base --options=Link \
            --groups "$baseGroupName/planta${floor}"
        "$vbox" modifyvm "pc${iClient}_pl${floor}" --groups "$baseGroupName/planta${floor}" --nic1 intnet --intnet1 "$nat_network"
    done
}

# Crear clientes de planta 0
create_floor0() {
    for iClient in "${pcs_pl0[@]}"; do
        "$vbox" clonevm "$basic_client_template" --name "${iClient}" --register --snapshot base --options=Link --groups "$baseGroupName/planta00"
        "$vbox" modifyvm "${iClient}" --groups "$baseGroupName/planta00" --nic1 intnet --intnet1 VCorp_pl0
    done
}

# -------------------------------
# Función de ayuda
# -------------------------------
show_help() {
    cat <<EOF
Usage $0 [options]
Options:
  -i, --import-ova          Import OVA file
  -r, --create-routers       Create routers
  -b, --create-floor0        Create clients in floor 0
  -p, --create-printers      Create printers
  -s, --create-servers       Create DMZ and private servers
  -f, --create-floors <N>    Create N clients per floor
  -d, --create-dev-servers   Create www and data servers for projects
  --rm-floor0                Remove floor 0 machines
  --rm-printers              Remove printers
  --rm-group <group_name>    Remove all VMs of a group
  --list                     List all VMs
  -h, --help                 Show this help
EOF
}

# -------------------------------
# Parseo de argumentos
# -------------------------------
if [ "$1" = "" ]; then
    set -- "-rbpsd -f 3"
fi

args=$(getopt -o bshirf:pd \
    --long create-floor0,rm-floor0,help,import-ova,create-routers,create-servers,create-dmz:,create-floors:,create-dev-servers,rm-group:,create-printers,rm-printers,list \
    --name "$0" -- "$@")
eval set -- "${args}"

while true; do
    case "$1" in
        -h|--help) show_help; shift ;;
        -b|--create-floor0) create_floor0; shift ;;
        --rm-floor0) rm_vms_of_group "/VCorp/planta00"; shift ;;
        -i|--import-ova) import_ova; shift ;;
        -r|--create-routers) create_routers; shift ;;
        -s|--create-servers) create_dmz "websrv,ftpsrv,streamingsrv"; create_private; shift ;;
        -p|--create-printers) create_printers; shift ;;
        --rm-printers) rm_printers; shift ;;
        -f|--create-floors) create_floor 01 "$2" VCorp_pl1; create_floor 02 "$2" VCorp_pl2; create_floor 03 "$2" VCorp_pl3; create_floor direccion "$2" VCorp_direccion; shift 2 ;;
        -d|--create-dev-servers) create_dev_servers prj1 planta01 VCorp_pl1; create_dev_servers prj2 planta01 VCorp_pl1; create_dev_servers prj3 planta02 VCorp_pl2; shift ;;
        --rm-group) rm_vms_of_group "$2"; shift 2 ;;
        --list) list_vms_names; shift ;;
        --) shift; break ;;
    esac
done

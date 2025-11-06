# NAS

#!/bin/bash
# Script NAS Samba autonome - Raspberry Pi 5
set -euo pipefail

# ===========================
#        CONFIG
# ===========================
DEVICE="/dev/sda1"
MOUNT_POINT="/mnt/nas"
LABEL="NAS_USB"
CONF_FILE="/etc/samba/smb.conf"
GROUP="smbusers"
LOG_FILE="/var/log/install_nas.log"

# ===========================
#        LOG SYSTEM
# ===========================
log() {
    local level="$1"; shift
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    echo "[$timestamp] [$level] $*" | tee -a "$LOG_FILE"
}

# ===========================
#       SANITY CHECKS
# ===========================
[ "$(id -u)" -ne 0 ] && { log ERROR "Ce script doit être exécuté en root"; exit 1; }

mkdir -p /var/log
echo "===== NAS Samba Auto-Reboot - $(date) =====" > "$LOG_FILE"

# ===========================
#      HELPER FUNCTIONS
# ===========================
disable_services() {
    for svc in bluetooth.service avahi-daemon.service; do
        if systemctl is-enabled "$svc" &>/dev/null; then
            systemctl disable "$svc"
            systemctl stop "$svc"
            log INFO "Service désactivé : $svc"
        fi
    done
}

wait_for_device() {
    local timeout=30 counter=0
    while [ ! -b "$DEVICE" ]; do
        log WARN "Périphérique $DEVICE non détecté, attente..."
        sleep 2
        counter=$((counter+2))
        [ $counter -ge $timeout ] && { log ERROR "Périphérique introuvable"; exit 1; }
    done
}

format_if_needed() {
    if ! blkid "$DEVICE" | grep -q "TYPE="; then
        log WARN "Formatage de $DEVICE en ext4…"
        mkfs.ext4 -F -L "$LABEL" "$DEVICE"
    fi
}

mount_device() {
    mkdir -p "$MOUNT_POINT"
    local uuid
    uuid=$(blkid -s UUID -o value "$DEVICE")

    mount -U "$uuid" "$MOUNT_POINT" 2>/dev/null || log INFO "Clé déjà montée"
    grep -q "$uuid" /etc/fstab || echo "UUID=$uuid $MOUNT_POINT ext4 defaults 0 2" >> /etc/fstab
}

create_shared_folders() {
    groupadd -f "$GROUP"

    for dir in public prive backup; do
        mkdir -p "$MOUNT_POINT/$dir"
        if [ "$dir" = "public" ]; then
            chmod 775 "$MOUNT_POINT/public"
        else
            chmod 770 "$MOUNT_POINT/$dir"
        fi
        chown root:"$GROUP" "$MOUNT_POINT/$dir"
    done

    log INFO "Dossiers public/prive/backup configurés"
}

install_samba() {
    apt update -y && apt install -y samba cifs-utils
    log INFO "Samba installé"
}

backup_samba_conf() {
    [ -f "$CONF_FILE" ] && cp "$CONF_FILE" "$CONF_FILE.bak.$(date +%F-%T)"
}

write_samba_conf() {
cat <<EOF > "$CONF_FILE"
[global]
   workgroup = WORKGROUP
   server string = Raspberry Pi NAS
   map to guest = Bad User
   dns proxy = no
   log file = /var/log/samba/log.%m
   max log size = 1000
   server role = standalone server

[public]
   path = $MOUNT_POINT/public
   browseable = yes
   read only = yes
   guest ok = yes
   force group = $GROUP
   create mask = 0664
   directory mask = 0775
   write list = @$GROUP

[prive]
   path = $MOUNT_POINT/prive
   browseable = yes
   writable = yes
   valid users = @$GROUP
   create mask = 0660
   directory mask = 0770

[backup]
   path = $MOUNT_POINT/backup
   browseable = yes
   writable = yes
   valid users = @$GROUP
   create mask = 0660
   directory mask = 0770
EOF
}

restart_samba() {
    systemctl restart smbd
    systemctl enable smbd
    systemctl is-active --quiet smbd && log INFO "Samba actif" || log ERROR "Échec Samba"
}

print_summary() {
    local ip
    ip=$(hostname -I | awk '{print $1}')
    echo "NAS Samba prêt : smb://$ip/"
}

# ===========================
#      SCRIPT EXECUTION
# ===========================
disable_services
wait_for_device
format_if_needed
mount_device
create_shared_folders
install_samba
backup_samba_conf
write_samba_conf
restart_samba
print_summary

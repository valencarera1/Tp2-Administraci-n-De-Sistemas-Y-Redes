# Servidor Web con VPN - WireGuard + Apache2
*TP2 - Administración de Sistemas y Redes*

## Integrantes
- Lucía Saint Martin
- Valentina Carera

## Entorno
- **Servidor**:
    - Notebook con Ubuntu Server 22.04 LTS
    - 10.0.0.1
- **Cliente**:
    - Dispositivo móvil Android con aplicación WireGuard
    - 10.0.0.2

## Arquitectura del Sistema:
El servidor está configurado con las siguientes características de seguridad:
- **Red pública**: Solo accesible el servicio VPN (WireGuard en puerto 51820/UDP).
- **Red privada (VPN)**: Accesibles SSH, Apache2 y demás servicios.
- **Firewall**: Configurado con UFW para denegar todo tráfico entrante excepto VPN.
- **Autenticación**: Usuarios requieren credenciales para acceder a sus sitios web.
- **Backup**: Sistema automatizado de respaldo de directorios de usuarios.


## Pasos
### Preparamos el entorno
1. Accedemos como administrador: 
    ```
    sudo -i
    ```

    Entramos como administrador a la terminal de host para poder tener los permisos necesarios durante todo el desarrollo del proyecto. Esto va a simplificar todos los comandos durante la configuración ya que no vamos a tener que anteponer sudo constantemente.

2. Actualizamos del Sistema Operativo:
    ```
    1. apt update
    2. apt upgrade -y
    ```
    Esto actualiza la lista de paquetes disponibles e instala actualizaciones de seguridad y mejoras del sistema y se confirma automáticamente las instalaciones.

3. Instalación de herramientas básicas:
    ```
    apt install -y net-tools ufw curl wget vim git htop openssh-server
    ```
    - **net-tools**: Comandos de red como ifconfig, netstat
    - **ufw**: Firewall simple de gestionar (Uncomplicated Firewall)
    - **curl y wge**t: Descarga de archivos desde la terminal
    - **vim**: Editor de texto avanzado
    - **git**: Control de versiones
    - **htop**: Monitor de procesos mejorado
    - **openssh-server**: Servidor SSH para administración remota

## Configuración de SSH
1. Instalación
    ```
    1. systemctl status ssh
    2. apt update
    3. apt install openssh-server -y
    4. systemctl enable ssh
    5. systemctl start ssh
    ```
    systemctl status ssh: Verifica si SSH está instalado y corriendo
    systemctl enable ssh: Configura SSH para iniciarse automáticamente al bootear
    systemctl start ssh: Inicia el servicio inmediatamente

2. Verificación del puerto SSH
    ```
    nano /etc/ssh/sshd_config
    ```
    Verificamos que el puerto sea el 22, el cual es el puerto predeterminado para SSH.

## Configuramos WireGuard (VPN)
1. Elegimos WireGuard porque es mas simple, moderno y más rapido que OpenVPN.

2. Instalación
    ```
    apt install wireguard -y
    ```

3. Creamos claves del servidor:
    ```
    mkdir -p /etc/wireguard
    cd /etc/wireguard
    wg genkey | tee server_private.key
    cat server_private.key | wg pubkey | tee server_public.key
    ```

    wg genkey: Genera una clave privada
    tee: Guarda la salida en archivo y la muestra en pantalla
    wg pubkey: Deriva la clave pública desde la privada

    Las claves privadas nunca las compartimos, solo intercambiamos claves públicas entre servidor y cliente.

4. Luego configuramos el WireGuard del servidor:
    ```
    nano /etc/wireguard/wg0.conf
    ```

    ``` otra opción:
    [Interface]
    Address = 10.0.0.1/16
    PrivateKey = <contenido_de_server_private.key>
    ListenPort = 51820

    PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
    PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

    [Peer]
    PublicKey = <clave_publica_del_cliente>
    AllowedIPs = 10.0.0.2/32
    ```

    **Address**: IP del servidor dentro de la VPN
    **PrivateKey**: Clave privada generada anteriormente (copiar contenido del archivo)
    **ListenPort**: Puerto UDP donde escucha WireGuard (51820 es estándar)
    **PostUp/PostDown**: Reglas iptables para enrutamiento (permiten que el tráfico VPN se enrute correctamente)
    **PublicKey (Peer)**: Clave pública del cliente móvil
    **AllowedIPs (Peer)**: IP asignada al cliente dentro de la VPN

    <!-- Nota: Reemplazar eth0 por el nombre de tu interfaz de red (usar ip a para verificar). -->

    ```como lo tenemos nosotras
    [Interface]
    Address = 10.0.0.1/16
    PrivateKey = server_private.key
    ListenPort = 51820

    [Peer]
    PublicKey = <clave pública que creó el WireGuard del cliente>
    AllowedIPs = 10.0.0.2/16
    Endpoint = <IP del dispositivo del cliente>:51820
    ```

    Aclaración para usaro nano:
    ```
    Ctrl + 0: Guardar
    Enter: Aceptar la ubicación del archivo, despues de fijarse que este bien
    Ctrl + X
    ```

5. Ajustar permisos de las claves
    ```
    chmod 600 /etc/wireguard/server_private.key
    chmod 600 /etc/wireguard/wg0.conf
    ```
    Las claves privadas deben ser legibles solo por root por seguridad.
6.  Iniciar WireGuard
    ```
    wg-quick up wg0
    wg show
    ```
    Esto levanta la interfaz de WireGuard y muestra el estado de las conexiones. La salida esperada sería el peer configurado y el endpoint.

    - Aclaración: en caso de modificación del archivo ``/etc/wireguard/wg0.conf`` luego de guardarlo en nano hay que reiniciarlo: ``wg-quick down wg0 && wg-quick up wg0 && wg show``

7. Habilitar el inicio automático:
    ```
    systemctl enable wg-quick@wg0
    ```
    Configura WireGuard para iniciarse automáticamente al bootear el servidor.

8. Para configurar el WireGuard del cliente:
    ```
    [Interface]
    Clave Privada: <clave privada que creó el WireGuard del cliente de la interfaz>
    Clave Pública: <clave pública que creó el WireGuard del cliente de la interfaz>
    Direcciones: 10.0.0.2/32
    Servidores DNS: 8.8.8.8

    [Peer]
    Clave Pública: server_public.key
    Keepalive persistente: 25 segundos
    IPs Permitidas: 10.0.0.1/32
    ```

9. Con WireGuard activamos el túnel que acabamos de crear y la conexión empieza automáticamente.
    - Aclarción: Para comprobar si anduvo, vemos si realizó el handshake y si el cliente está recibiendo y enviando datos al servidor.
    
## Configuramos el Firewall
1. Ver estado actual y reseteamos las reglas
    ```
    ufw status verbose
    ufw reset
    ```
    Lo reseteamos para porder hacer una configuración limpia.

2. Bloqueamos todo lo entrante y permitimos lo saliente:
    ```
    ufw default deny incoming
    ufw default allow outgoing
    ```

3. Permitir WireGuard (VPN)
    ```
    ufw allow 51820/udp
    ```
    Este es el ÚNICO puerto accesible desde internet. Todo lo demás será accesible solo a través de la VPN.

4. Permitimos SSH:
    ```
    ufw allow in on wg0 to any port 22
    ```
    Solo acepta conexiones SSH que vengan desde la interfaz WireGuard.
    Esto hace que SSH sea inaccesible desde internet, solo desde la VPN.

5. Permitir Apache solo desde la VPN
    ```
    ufw allow in on wg0 to any port 80
    ufw allow in on wg0 to any port 443
    ```
    HTTP (80) y HTTPS (443) solo accesibles desde la VPN.

6. Permitir tráfico interno de la VPN
    ```
    ufw allow in on wg0
    ufw allow out on wg0
    ```
    Permite comunicación libre entre dispositivos dentro de la VPN.

7. Activar el firewall
    ```
    ufw enable
    ```

## Configuramos Apache2
1. Instalación
    ```
    apt install apache2 -y
    ```
2. Habilitar el módulo userdir
    ```
    a2enmod userdir
    systemctl restart apache2
    ```

3. Verificar inicio automático
    ```
    bashsystemctl enable apache2
    systemctl is-enabled apache2
    ```

4. Crear directorios personales para los usuarios y configurarlos
    ```
    mkdir -p /home/valen/public_html
    chown -R valen:valen /home/valen/public_html
    chmod 755 /home/valen/public_html
    a2enmod userdir
    http://localhost/~valen
    ```

    - **7 (owner)**: Lectura, escritura, ejecución
    - **5 (group)**: Lectura y ejecución
    - **5 (others)**: Lectura y ejecución

5. Configurar módulo userdir
    ```
    nano /etc/apache2/mods-enabled/userdir.conf
    ```
    
    ```
    <IfModule mod_userdir.c>
        UserDir public_html
        UserDir enabled *

        <Directory /home/*/public_html>
            AllowOverride All
            Options Indexes FollowSymLinks MultiViews
            Require all granted
        </Directory>
    </IfModule>
    ```

    - UserDir enabled **: Habilita userdir para todos los usuarios
    - **AllowOverride All**: Permite archivos .htaccess (necesario para autenticación)
    - **Options Indexes**: Muestra listado de archivos si no hay index.html
    - **Require all granted**: Permite acceso (la autenticación se maneja con .htaccess)

6. Permisos de directorios HOME
    ```
    chmod 755 /home
    chmod 755 /home/*
    chmod 755 /home/*/public_html
    ```
    Apache necesita permisos de lectura y ejecución para navegar hasta public_html.

7. Reiniciar Apache
    ```
    systemctl restart apache2
    ```

8. Creamos los usuarios como ejemplo:
    ```
    adduser valenlu
    ```

    Luego, creamos los directorios public_html para cada usuario:
    ```
    sudo -u valenlu mkdir -p /home/valenlu/public_html
    sudo -u valenlu chmod 755 /home/valenlu/public_html
    sudo chmod 755 /home/valenlu
    ```

    Por último modificamos el archivo: `nano /home/valenlu/public_html/index.html` para cada usuario:
    ```
        
    ```

    

9. Modificamos el archivo: ``nano /var/www/html/index.html``
    ```
    <!DOCTYPE html>
    <html lang="es">
    <head>
    <meta charset="UTF-8">
    <title>Usuarios del servidor</title>
    <style>
        body {
        font-family: Arial, sans-serif;
        background: #f9f9f9;
        text-align: center;
        padding: 40px;
        }
        ul {
        list-style: none;
        padding: 0;
        }
        li {
        margin: 10px 0;
        }
        a {
        text-decoration: none;
        color: #0077cc;
        font-weight: bold;
        }
    </style>
    </head>
    <body>
    <h1>Bienvenido al servidor Apache</h1>
    <p>Seleccioná un usuario para ver su sitio:</p>
    <ul>
        <li><a href="/~valen/">valen</a></li>
    </ul>
    </body>
    </html>
    ```
    - Aclaración: Este archivo es la pagina principal de Apache y la modificamos para que salga lo que nosotras queremos y no la pagina default de Apache2

    8. Abrimos http://<IP servidor>/~valen/ desde el cliente y vemos que salio nuestra pagina.
    *ADJUNTAR IMAGEN*


## Configuramos la autenticacion de los usuarios
1. Configurar archivo .htaccess
    ```
    nano /home/valen/public_html/.htaccess
    ```

    ```
    apacheAuthType Basic
    AuthName "Área restringida - Identificate"
    AuthUserFile /home/valen/.htpasswd
    Require valid-user
    ```
    - **AuthType Basic**: Autenticación HTTP básica
    - **AuthName**: Mensaje que aparece en el popup de login
    - **AuthUserFile**: Ubicación del archivo con contraseñas hasheadas
    - **Require valid-user**: Requiere credenciales válidas para acceder

3. Creamos el archivo de contraseñas
    ```
    htpasswd -c /home/valen/.htpasswd valen
    ```
    - **-c**: Crea un nuevo archivo
    Solicitará ingresar y confirmar la contraseña
    La contraseña se almacena hasheada

    Para agregar más usuarios al mismo archivo (sin -c):
    ```
    bashhtpasswd /home/valen/.htpasswd otrousuario
    ```
    
4. Una vez hecho esto reiniciamos apache
    ```
    systemctl restart apache2
    ```

5. Probar autenticación

    1. Nos conectamos a la VPN desde el cliente
    2. Abrimos navegador y accedemos a http://10.0.0.1/~valen/
    3. Aparece un cartel solicitando usuario y contraseña
    4. Ingresamos credenciales creadas con htpasswd


## Sistema de Backup Automático
1. Crear script de backup
    ```
    nano /usr/local/bin/backup_users.sh
    ```

    ```
    bash#!/bin/bash

    # Configuración
    BACKUP_DIR="/var/backups/user_homes"
    DATE=$(date +%Y%m%d_%H%M%S)
    USERS_HOME="/home"
    RETENTION_DAYS=30

    # Crear directorio de backups si no existe
    mkdir -p "$BACKUP_DIR"

    # Realizar backup de cada usuario
    for USER_DIR in "$USERS_HOME"/*; do
        if [ -d "$USER_DIR" ]; then
            USERNAME=$(basename "$USER_DIR")
            
            # Crear directorio para el usuario
            mkdir -p "$BACKUP_DIR/$USERNAME"
            
            # Realizar backup con tar y gzip
            tar -czf "$BACKUP_DIR/$USERNAME/backup_${USERNAME}_${DATE}.tar.gz" \
                -C "$USERS_HOME" "$USERNAME" 2>/dev/null
            
            if [ $? -eq 0 ]; then
                echo "[$(date)] Backup exitoso de $USERNAME"
            else
                echo "[$(date)] Error en backup de $USERNAME" >&2
            fi
        fi
    done

    # Eliminar backups antiguos
    find "$BACKUP_DIR" -name "backup_*.tar.gz" -mtime +$RETENTION_DAYS -delete

    echo "[$(date)] Proceso de backup completado"
    ```

    Este script:
    - Crea backups comprimidos de cada directorio de usuario
    - Incluye fecha y hora en el nombre del archivo
    - Elimina automáticamente backups con más de 30 días
    - Registra éxitos y errores en el log


2. Dar permisos de ejecución
    ```
    chmod +x /usr/local/bin/backup_users.sh
    ```

3. Probar el script manualmente
    ```
    /usr/local/bin/backup_users.sh
    ls -lh /var/backups/user_homes/
    ```
    Se ven carpetas con los nombres de usuarios y archivos .tar.gz.

4. Automatizar con cron
    ```
    crontab -e
    ```
    ```
    0 3 * * * /usr/local/bin/backup_users.sh >> /var/log/backup_users.log 2>&1
    ```
    - 0 3 * * *: Todos los días a las 3:00 AM
    - **>> /var/log/backup_users.log**: Guarda la salida en un log
    - **2>&1**: Redirige errores también al log

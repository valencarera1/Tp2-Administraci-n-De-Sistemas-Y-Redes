# Tp2-Administración-De-Sistemas-Y-Redes
## Integrantes:
- Lucía Saint mArtin
- Valentina Carera

## Entorno:
- Servidor: Notebook Ubuntu Server 22.04 LTS
- Cliente: Celular desde Wireguard
## Pasos:
### Preparamos el entorno:
1. Entramos como administrador a la terminal de host para poder tener los permisos necesarios durante todo el desarrollo del proyecto: ``sudo -i``

2. Actualizamos el SO:
    ```
    1. apt update
    2. apt upgrade
    ```

3. Finalmente instalamos las herramientas que vamos a necesitar durante el proyecto:
    ```
    1. apt install -y net-tools ufw curl wget vim git
    2. apt install -y htop openssh-server
    ```

## Preparamos el servidor:
### 1. Configuramos SSH:
    ```
    1. systemctl status ssh
    2. apt update
    3. apt install openssh-server -y
    4. systemctl enable ssh
    5. systemctl start ssh
    ```

### 2. Instalación de WireGuard:
- Decidimos utilizar Wire Guard porque en nuestra investigación nos pareció lo más sencillo y fácil de aprender: apt install wireguard
    1. Creamos claves públicas y privadas del servidor:
        ```
        1. mkdir -p /etc/wireguard
        2. cd /etc/wireguard
        3. wg genkey | tee server_private.key
        4. cat server_private.key | wg pubkey | tee server_public.key
        ```

    2. Las claves públicas y privadas del celular las genera WireGuard automáticamente.

    3. Luego configuramos el wg0 del servidor:
    ```
    nano /etc/wireguard/wg0.conf
    /etc/wireguard/wg0.conf
    ```

    ```
    [Interface]
    Address = 10.0.0.1/16
    PrivateKey = server_private.key
    ListenPort = 51820

    [Peer]
    PublicKey = <clave pública que creó el WireGuard del cliente>
    AllowedIPs = 10.0.0.2/16
    Endpoint = <IP del dispositivo del cliente>:51820
    ```

    4. Guardamos el archivo en nano: ``Ctrl + 0 - Enter - Ctrl + X``

    5. Levantamos wg0 y comprobamos que se haya guardado correctamente:
    ```
    wg-quick up wg0 y wg show
    ```

    - Aclaración: en caso de modificación del archivo ``/etc/wireguard/wg0.conf`` luego de guardarlo en nano hay que reiniciarlo: ``wg-quick down wg0 && wg-quick up wg0 && wg show``

    6. Ni idea que hace esto: ``systemctl enable wg-quick@wg0``

    7. Para configurar el WireGuard del cliente:
    
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

    8. Con WireGuard creamos el túnel y se activa la conexión automáticamente.
    - Aclarción: Para comprobar si anda, vemos que realizó el handshake y que el cliente está recibiendo y enviando datos al servidor.
    
### 3. Configuramos el Firewall
- AAAA
    1. Ver estado actual:
    ```
    sudo ufw status verbose
    ```
    
    2. Reseteamos reglas anteriores:
    ```
    sudo ufw reset
    ```

    3. Bloqueamos todo lo entrante y permitimos lo saliente:
    ```
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    ```

    4. Permitimos SSH:
    ```
    sudo ufw allow 22/tcp
    ```
    - Aclaración: Antes de esta paso verificamos en que puerto nuestro SSH estaba: ``sudo nano /etc/ssh/sshd_config``
    5. Permitir el puerto de WireGuard (51820/UDP)
    ```
    sudo ufw allow 51820/udp
    ```
    6. Este paso no es completamente necesario pero permitimos el tráfico interno del túnel VPN:
        - WireGuard usa una red interna (por ejemplo 10.8.0.0/24). Para permitir tráfico desde esa red:
    ```
    sudo ufw allow in on wg0
    sudo ufw allow out on wg0
    ```

    7. Por ultimo activamos el firewall:
    ```
    sudo ufw enable
    ```


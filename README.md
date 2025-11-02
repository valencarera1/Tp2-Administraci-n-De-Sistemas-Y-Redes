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

### 2. Configuramos WireGuard:
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

### 3. Configuramos Apache2:
1. Instalamos Apache:
    ```
    sudo apt install apache2 -y
    ```
2. Habilitar el módulo userdir:
    ```
    sudo a2enmod userdir
    sudo systemctl restart apache2
    ```
3. Crear directorios personales para los usuarios y configurarlos:
    ```
    mkdir -p /home/valen/public_html
    chown -R valen:valen /home/valen/public_html
    chmod 755 /home/valen/public_html
    sudo a2enmod userdir
    http://localhost/~valen
    ```
4. Modificar el archivo /etc/apache2/mods-enabled/userdir.conf:
    ```
    UserDir public_html
    UserDir enabled *
    <Directory /home/*/public_html>
        AllowOverride All
        Options Indexes FollowSymLinks MultiViews
        Require all granted
    </Directory>
    ```
5. Asegurarse de que Apache tenga acceso a todos los archivos con los
siguientes comandos:
    ```
    sudo chmod 755 /home
    sudo chmod 755 /home/*
    sudo chmod 755 /home/*/public_html
    ```

6. Reiniciar Apache:
    ```
    sudo systemctl restart apache2
    ```
<!-- • Crear una página de inicio con enlaces a los usuarios:
<ul>
<li><a href="/~user/">user</a></li>
</ul> -->

7. Modificamos el archivo: ``sudo nano /var/www/html/index.html``
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


### 3. Configuramos la autenticacion de los usuarios:
1. Abrimos la carpeta del usuario:
    ```
    cd /home/valen/public_html
    nano .htaccess
    ```
2. Ahi dentro configuramos que el usuario tenga que autenticarse:
    ```
    AuthType Basic
    AuthName "Acceso restringido"
    AuthUserFile /home/valen/.htpasswd
    Require valid-user
    ```
3. Creamos el archivo de contraseñas:
    ```
    sudo htpasswd -c /home/valen/.htpasswd valen
    ```
    - Aclaración: Luego de este paso nos pidio ingresar una contraseña para el usuario y la almacenó.
    
4. Una vez hecho esto reiniciamos apache: 
5. Finalemente verificamos que anda:
    ```
    1. Entramos a *http......*
    2. Ingresamos a valen
    3. Nos va a pedir la contraseña y la ingresamos
    4. A
    ```


<!-- 


sudo nano /usr/local/bin/generar_home.sh
sudo chmod +x /usr/local/bin/generar_home.sh
<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8">
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
  <h1>Usuarios disponibles</h1>
  <p>Seleccioná un usuario para ver su sitio:</p>
  <ul>
    <li><a href="/~valen/">valen</a></li>
    <li><a href="/~lucia/">lucia</a></li>
  </ul>
</body>
</html>

 -->




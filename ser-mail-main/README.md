# Práctica: Instalación y configuración de un servicio de email

## Parte 1: Servidor DNS

Ncestiamos montar un servidor DNS para que los servidores de correo sepan a donde deben enviar el correo destinado a nuestro dominio. Para ello, utilizaremos lo mismo que hicimos en la primera UF.

Instalamos bind9:

> `sudo apt install bind9`

Modificamos el fichero `/etc/bind/named.conf.local` para añadir nuestra nuestro fichero de zona, que también crearemos.

En la práctica primera tienes instrucciones más exactas sobre cómo hacer esto. Además, aquí tienes ejemplos.

Comprueba la configuración con:

> - `sudo named-checkconf /etc/bind/named.conf.local`
>
> - `sudo named-checkzone test.com /etc/bind/db.test.com` 
>
> - `nslookup -type=MX test.com localhost`

Deberás configurar tu servidor para utilizar este servidor DNS como su servidor DNS para poder realizar correctamente el resto de la práctica. Opcionalmente, también deberás configurar este servidor DNS en las máquinas donde pruebes a utilizar un programa de gestión de correo.

## Parte 2: Mail Transfer Agent (Postfix)

### Configuración básica

El primer paso de esta parte será instalar y configurar Postfix:

> `sudo apt install postfix`

El instalador de Postfix nos hará varias preguntas. La primera es cómo queremos configurar nuestro servidor, a elegir entre varias opciones: *Internet Site* será el más parecido a lo que buscamos, aunque no cumpliremos todos los requisitos para hacerlo bien, faltándonos el control de un dominio real o un [FQDN (_Fully Qualified Domain Name_)](https://www.ionos.es/digitalguide/dominios/gestion-de-dominios/fully-qualified-domain-name/).

A continuación nos pedirá el nombre de nuestro dominio. Si queremos mandar un mail desde las cuentas paco@test.com, maite@test.com, deberemos escribir `test.com`.

Durante la instalación del servidor, el instalador se encargará de crear un fichero de configuración básico en `/etc/postfix/main.cf`. Para modificarlo, vamos a hacer varias modificaciones sobre él utilizando la herramienta `postconf`. Esta herramienta recibe un parámetro, su valor y actuliza el fichero de configuración:

> `sudo postconf 'home_mailbox= Maildir/'`
> `sudo postconf 'mydomain=test.com'`
> `sudo postconf 'myorigin=$mydomain'`

El primer comando utiliza Maildir como formato para guardar el correo. Este formato es un sustituto de mbox, el que utiliza por defecto, y que se está abandonando porque guarda todo el correo en un solo fichero, lo que complica su gestión. Maildir utiliza un directorio en el que crea diferentes ficheros para cada correo.

El segundo y tercer comando sirven para que los correos electrócnicos salientes se firmen con el nombre@test.com, ya que por defecto los firmará como nombre@nombre_de_la_máquina, lo cual rara vez es lo que queremos (salvo que estemos montando un servicio de correo MUY local y aislado de todo lo demás).

Cuando hagamos modificaciones al fichero de configuración, deberemos recargar la configuración de postfix:

> `sudo postfix reload`

O el archiconocido:

> `sudo systemctl reload postfix`

### Probar la configuración

Llegados a este punto, ya podemos probar el envío de correos por nuestra parte. Lo más probable si intentamos mandar un mail a una dirección de mail decente, es que no nos deje, por medidas antispam. Para probar la práctica, por lo tanto, lo mejor será mandarlo a un servicio indecente, como [10 minute mail](https://10minutemail.com/). Esta vez crea una dirección de correo temporal que podrás utilizar para tus pruebas.

Para mandar un mail, te recomiendo utilizar la herramienta s-nail:

> `sudo apt install s-nail`

Es un programa para línea de comandos que nos permitirá mandar y leer mails.

Deberás configurarlo mínimanmente para poder usarlo, modificando el fichero '/etc/s-nail.rc' y añadiendo las siguientes líneas:

- `set empystart`, para permitir que se ejecute incluso con la bandeja de entrada vacía. Posiblemente ya esté puesta en el fichero.
- `set folder=Maildir`para que sepa dónde encontrar el mail.
- `set record=+sent` para guardar una copia de los mails enviados.

Para enviar un mail, lo más sencillo es utilizar el siguiente comando:

> `echo "Cuerpo del mensaje" | s-nail -s "Asunto" dirección_destion@dominio.com`

### Envíos cifrados

Vamos, ahora, con los certificados para lograr conexiones seguras.

Primero, deberemos generar las claves. Hay muchos algoritmos y configuraciones válidas, así que tu comando no tendría por qué ser igual a este:

> `sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/mail.key -out /etc/ssl/certs/mail.crt`

Creados los certificados, deberemos especificar la ruta a postfix. De nuevo, podemos modificar el fichero `/etc/postfix/main.cf` o podemos utilizar `postconf`. En cualquiera de los dos casos, deberemos prestar atención a tres parámetros:

- `smtpd_tls_cert_file` con la ruta al fichero de certificado.
- `smtpd_tls_key_file` con la ruta a la clave privada.
- `smtpd_tls_security_level` que nos permitirá establecer cuánta seguridad utilizaremos. Las opciones más básicas son _none_ para no encriptar las conexiones, _may_ para hacerlo solo si el otro servidor lo permite y _encrypt_ para forzar la encriptación. Puede verse un listado completo de la configuración en los [docs de Postfix](http://www.postfix.org/TLS_README.html#client_tls_encrypt)

### Configurar más usuarios

Hay dos maneras de crear cuentas de usuario.

1. Por un lado, podemos crear nuevos usuarios de nuestro SO, teniendo cada uno de ellos su propia cuenta en Postfix automáticamente. Esto sería la manera básica y estándar. `sudo add-user`...
2. Por otro lado, podemos utilizar alias de cuenta, para que varias direcciones de mail entrengen correo al mismo usuario de nuestro SO. Para eso, deberemos configurar un fichero virtual_alias en Postfix. Puede leer más al respecto en este [tutorial](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-postfix-on-ubuntu-20-04-es#paso-2-cambiar-la-configuracion-de-postfix) y en la [documentación oficial](https://www.postfix.org/postconf.5.html#virtual_alias_maps).

## Parte 3: Dovecot

Dovecot viene en los repositorios de Ubuntu dividido en varios paquetes. Nosotros necesitaremos instalar solo algunos. La manera más directa es la siguiente:

> `sudo apt install dovecot-imapd dovecot-pop3d`

En una instalación real podríamos querer más, como:
- dovecot-sieve (filtros de mail), 
- dovecot-solr (búsqueda en el texto de los mensajes),
- dovecot-antispam (filtros antispam).

Llegados a este punto, deberemos empezar con las configuraciones que deseamos aplicar. Dovecot divide sus ficheros de configuración en varios. El principal es `/etc/dovecot/dovecot.conf`, pero normalmente querrás tocar los ficheros incluidos en el directorio `/etc/dovecot/conf.d`, habiendo uno para cada familia de parámetros configurables.

Vamos con ello.

## Utilizar Maildir

Le habíamos dicho a Postfix que usara el formato Maildir y a Dovecot deberemos decirle otro tanto.

Editamos `/etc/dovecot/conf.d/10-mail.conf` y modificamos la línea referente a `mail_location` (posiblemente puedas comentar y descomentar):

> `mail_location = maildir:~/Maildir`
>
> `# mail_location = mbox:~/mail:INBOX=/var/mail/%u`

## Configuración de SSL

Editamos `/etc/dovecot/conf.d/10-ssl.conf` y modificamos las líneas de `ssl_cert` y `ssl_key` con los valores correctos (según dónde los hayamos guardado antes).

El caracter `<` después del igual no es una errata y debes dejarlo. En los ficheros de configuración de Dovecot sirve para especificar que ese parámetros cogerá como valor el contenido del fichero que se detalla a continuación. Si solo pusieras los igual, utilizaría como certificados las rutas a los ficheros, no su contenido.

## Autentificación

Impedir la autentificación por texto plano se puede lograr por dos maneras. Citando de la documentación oficial de Dovecot:

> There are a couple of different ways to specify when SSL/TLS is required:
>
>   **ssl=yes and disable_plaintext_auth=no**: SSL/TLS is offered to the client, but the client isn’t required to use it. The client is allowed to login with plaintext authentication even when SSL/TLS isn’t enabled on the connection. This is insecure, because the plaintext password is exposed to the internet.
>
>   **ssl=yes and disable_plaintext_auth=yes**: SSL/TLS is offered to the client, but the client isn’t required to use it. The client isn’t allowed to use plaintext authentication, unless SSL/TLS is enabled first. However, if non-plaintext authentication mechanisms are enabled they are still allowed even without SSL/TLS. Depending on how secure they are, the authentication is either fully secure or it could have some ways for it to be attacked.
>
>   **ssl=required**: SSL/TLS is always required, even if non-plaintext authentication mechanisms are used. Any attempt to authenticate before SSL/TLS is enabled will cause an authentication failure. Note that this setting is unrelated to the STARTTLS command - either implicit SSL/TLS or STARTTLS command is allowed.

Lo ideal es, por tanto, hacer un **ssl=required**.

# Probándolo todo

Lo más sencillo para probar todo es configurar un cliente de correo electrónico (como Thunderbird) en otra máquina y probra a mandar/recibir correos entre cuentas creadas en el servidor.

Para poder escribir la dirección del servidor en lugar de la IP, deberíamos configurar nuestra máquina para utilizar el DNS que configuramos al principio del todo. Podemos evitar ese paso si directamente utilizamos la IP del servidor de correo.
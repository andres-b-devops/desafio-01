# Gestión de usuarios en Linux usando Pipeline Declarativo

***Por: Andrés Burbano***

---
El presente documento muestra los pasos detallados para la implementación de un *pipeline declarativo* en Jenkins cuyo propósito es la gestión de usuarios dentro de un sistema Linux.

---
## Tabla de contenido
1. [Introducción](#1-introducción)
2. [Instalación](#2-instalación)
   - [Requisitos previos](#21-requisitos-previos)
   - [Job para la Creación de Usuarios](#22-job-para-la-creación-de-usuarios)
   - [Job para la Eliminación de un usuario](#23-job-para-la-eliminar-un-usuario)
---

## 1. Introducción

Este proyecto tiene como objetivo **facilitar la tarea de creación y eliminación de usuarios** en un sistema Linux mediante el uso de Jenkins con script declarativos. Está diseñado para **ser rápido**, **seguro** y **fácil de usar** para un operador de seguridad. 

Se tendrán en cuenta estos datos de entrada:
- **Login (username)**: identificador único que se compone del nombre y el apellido
- **Nombre y Apellido**: Nombre y Apellido del usuario
- **Departamento**: Este es el grupo que corresponde al área del usuario. Estos grupos son :```contabilidad```,```finanzas```,```tecnologia```

### Características:

#### 1. Job que permite crear usuarios:

- ✅ Se genera un password temporal que se asigna al usuario y que luego el usuario final deberá cambiar en el primer inicio de sesión.
- ✅ Se mostrará el password al operador el cuál se puede copiar y enviar por email al usuario final (se genera también una plantilla HTML con un saludo de bienvenida y el password creado)

#### 2. Job que permite eliminar usuarios:

- ✅ Recibe como parámetro el id del usuario (username)
- ✅ Eliminará tanto el ***grupo*** del usuario como el directorio ***home*** del mismo
---

## 2. Instalación
### 2.1. Requisitos previos:

**1. Instalación de Jenkins**: asegurarse de que Jenkins esté instalado en la máquina Linux donde se ejecutarán los jobs y se deseen crear/eliminar los usuarios.
**2. Privilegios de sudo:** el usuario que ejecuta trabajos de Jenkins debe tener privilegios de sudo para agregar y eliminar usuarios sin que se le solicite una contraseña.
Se debe configurar el archivo ```/etc/sudoers```  ejectuanto ```sudo visudo```, agregar lo siguiente:
```bash
jenkins ALL=(ALL:ALL) NOPASSWD:ALL
```

**3. Crear grupos de Departamentos**:
Se debe crear en el sistema Linux los siguientes grupos: ```contabilidad```,```finanzas```,```tecnologia``` con el siguiente código:
```bash
sudo groupadd contabilidad
sudo groupadd finanzas
sudo groupadd tecnologia
```
Verificar que los grupos fueron creados con la siguiente instrucción:  ```tail /etc/group```, debe mostrar una salida como la siguiente:
```bash
contabilidad:x:1004:
finanzas:x:1005:
tecnologia:x:1006:
```
**5. Instalar paquetes útiles**
- **Finger**: Muestra datos del usuario
```bash
sudo apt install finger -y

USO:

finger nombreusuario

```

### 2.2. Job para la Creación de Usuarios
#### Paso 1: Crear el Pipeline Declarativo
Crear un nuevo proyecto de tipo "Pipeline" en Jenkins y asignarle un nombre, por ejemplo, "CrearUsuario".
#### Paso 2: Definir los Parámetros de Entrada
En la sección de configuración del proyecto, en "This project is parameterized", agregar los siguientes parámetros:

* **String Parameter**: ```USERNAME```
* **String Parameter**: ```NOMBRE```
* **String Parameter**: ```APELLIDO```
* **Choice Parameter**: ```DEPARTAMENTO``` con opciones:
  * ```contabilidad```
  * ```finanzas```
  * ```tecnologia```


#### Paso 3: Escribir el Jenkinsfile

En la sección "Pipeline", seleccionar "Pipeline script" y pegar el siguiente script:

```bash
pipeline {
    agent any
    stages {
        stage('Validar Parámetros') {
            steps {
                script {
                    if (!USERNAME || !NOMBRE || !APELLIDO || !DEPARTAMENTO) {
                        error "Todos los parámetros deben ser proporcionados."
                    }
                }
            }
        }
        stage('Crear Usuario') {
            steps {
                script {
                    // Generar password temporal
                    def tempPassword = UUID.randomUUID().toString().replaceAll('-', '').take(8)
                    
                    // Comandos para crear usuario y asignar al grupo
                    sh """
                        sudo useradd -m -c "${NOMBRE} ${APELLIDO}" -g ${DEPARTAMENTO} -s /bin/bash ${USERNAME}
                        echo "${USERNAME}:${tempPassword}" | sudo chpasswd
                        sudo chage -d 0 ${USERNAME}
                    """
                    
                    // Guardar el password temporal en una variable
                    env.TEMP_PASSWORD = tempPassword
                }
            }
        }
        stage('Mostrar Información') {
            steps {
                echo "Usuario creado exitosamente."
                echo "Nombre: ${NOMBRE} ${APELLIDO}"
                echo "Username: ${USERNAME}"
                echo "Password temporal: ${env.TEMP_PASSWORD}"
            }
        }
        stage('Generar Plantilla HTML') {
            steps {
                script {
                    def htmlContent = """
                    <html>
                    <body>
                        <h1>Bienvenido, ${NOMBRE} ${APELLIDO}</h1>
                        <p>Su usuario ha sido creado con el nombre de usuario: <strong>${USERNAME}</strong></p>
                        <p>Su password temporal es: <strong>${env.TEMP_PASSWORD}</strong></p>
                        <p>Por favor, cambie su password en el primer inicio de sesión.</p>
                    </body>
                    </html>
                    """
                    writeFile file: 'bienvenida.html', text: htmlContent
                    echo "Plantilla HTML generada: bienvenida.html"
                }
            }
        }
        stage('Mostrar Plantilla en Consola') {
            steps {
                script {
                    // Leer y mostrar la plantilla en la consola
                    def htmlOutput = readFile 'bienvenida.html'
                    echo "Plantilla Generada:"
                    echo htmlOutput
                }
            }
        }
    }
    post {
        success {
            echo "Usuario: ${USERNAME}"
            echo "Password: ${env.TEMP_PASSWORD}"
            echo "Departamento: ${DEPARTAMENTO}"
        }
        aborted {
            echo "El proceso fue abortado. El usuario no fue creado."
            sh "sudo userdel -r ${USERNAME}"
        }
        failure {
            echo "Hubo un error en el proceso. Eliminando usuario si fue creado."
            sh "sudo userdel -r ${USERNAME} || true" // Evita errores si el usuario no existe
        }
    }
}
```

#### Paso 4: Ejecutar el Job

Darle click al botón play que dice "Build with Parameters". Se solicitarán los parámetros. Una vez completados, el pipeline creará el usuario y mostrará en pantalla el password temporal y la plantilla HTML.

#### Paso 5: Verificación de creación del usuario en Linux con su respectivo departamento.

Comprobar que se haya creado el usuario en la terminal de Linux
* Con ***Finger***:
```bash
finger nombreusuario

// Resultado esperado:

        Login: nombreusuario    			Name:  Nombre Usuario
        Directory: /home/nombreusuario        	Shell: /bin/bash
        Never logged in.
        No mail.
        No Plan.
```
* Revisando el archivo ```/etc/passwd``` debe aparece el usuario que se acaba de crear:
```bash
cat /etc/passwd | grep nombreusuario

// Resultado esperado:

        nombreusuario:x:1002:1005:Nombre Usuario:/home/nombreusuario:/bin/bash
```
Comprobar que se haya creado el directorio home del usuario en la terminal de Linux

* Revisando el listado de directorios de ```/home```:
```bash
ls -alh /home

// Resultado esperado:

        drwxr-xr-x  5 root          root      4.0K Nov 17 00:25 .
        drwxr-xr-x 22 root          root      4.0K Nov 16 10:48 ..
        drwxr-x---  2 nombreusuario finanzas  4.0K Nov 17 00:09 nombreusuario
        drwxr-x---  9 ubuntu        ubuntu    4.0K Nov 14 17:48 ubuntu
```
* Inspeccionar que los grupos pertenecientes con el usuario
(con el comando ```id nombreusuario```)

```bash
id nombreusuario

// Resultado esperado:

        uid=1002(nombreusuario) gid=1005(finanzas) groups=1005(finanzas)

        drwxr-xr-x  5 root          root      4.0K Nov 17 00:25 .
        drwxr-xr-x 22 root          root      4.0K Nov 16 10:48 ..
        drwxr-x---  2 nombreusuario finanzas  4.0K Nov 17 00:09 nombreusuario
        drwxr-x---  9 ubuntu        ubuntu    4.0K Nov 14 17:48 ubuntu
```

* Verificar que se pueda ingresar con la contraseña temporal y que solicite cambiarla en el primer inicio de sesión

  1.	Impersonar el usuario con el comando: ```sudo su – nombreusuario```
  2.	Ingresar la contraseña generada en el pipeline
  3.	Ingresar la nueva contraseña
  4.	Repetir la nueva contraseña 
  5.	Verificar que se haya pasado al nuevo usuario en la terminal (nombreusuario@devopsbootcamp)

```bash
sudo su - nombreusuario

// Resultado esperado:

      You are required to change your password immediately (administrator enforced).
      Changing password for nombreusuario.
      Current password:
      New password:
      Retype new password:
      
      nombreusuario@devopsbootcamp:~$

```

*	Verificar que esté dentro del directorio home asociado al usuario con el comando ```pwd```
```bash

nombreusuario@devopsbootcamp:~$ pwd

// Resultado esperado:

      /home/nombreusuario

```
* Crear un archivo de texto, luego verificar que el propietario sea el usuario “nombreusuario” y el grupo sea “finanzas”:
 
```bash
nombreusuario@devopsbootcamp:~$ echo "Hola mundo!" >> saludo.txt
nombreusuario@devopsbootcamp:~$ cat saludo.txt

// Resultado esperado:

      Hola mundo!

nombreusuario@devopsbootcamp:~$ ls -alh

// Resultado esperado:

      total 24K
      4.0K drwxr-x--- 2 nombreusuario finanzas 4.0K Nov 18 23:41 .
      4.0K drwxr-xr-x 5 root        root     4.0K Nov 18 23:26 ..
      4.0K -rw-r--r-- 1 nombreusuario finanzas  220 Mar 31  2024 .bash_logout
      4.0K -rw-r--r-- 1 nombreusuario finanzas 3.7K Mar 31  2024 .bashrc
      4.0K -rw-r--r-- 1 nombreusuario finanzas  807 Mar 31  2024 .profile
      4.0K -rw-r--r-- 1 nombreusuario finanzas   12 Nov 18 23:41 saludo.txt

```
---

### 2.3. Job para la Eliminar un Usuario
#### Paso 1: Crear el Pipeline Declarativo
Crear un nuevo proyecto de tipo "Pipeline" en Jenkins y asignarle un nombre, por ejemplo, "EliminarUsuario".
#### Paso 2: Definir los Parámetros de Entrada
En la sección de configuración del proyecto, en "This project is parameterized", agregar el siguiente parámetro:
* **String Parameter**: ```USERNAME```
#### Paso 3: Escribir el Jenkinsfile
En la sección "Pipeline", seleccionar "Pipeline script" y pegar el siguiente script:
```bash
pipeline {
    agent any
    stages {
        stage('Validar Parámetro') {
            steps {
                script {
                    if (!USERNAME) {
                        error "El username debe ser proporcionado."
                    }
                }
            }
        }
        stage('Eliminar Usuario') {
            steps {
                script {
                    sh """
                        sudo userdel -r ${USERNAME}
                    """
                }
            }
        }
        stage('Confirmación') {
            steps {
                echo "Usuario ${USERNAME} eliminado exitosamente junto con su directorio home."
            }
        }
    }
    post {
        success {
            echo "El usuario ${USERNAME} fue eliminado exitosamente."
        }
        aborted {
            echo "El proceso fue abortado. No se eliminó el usuario ${USERNAME}."
        }
        failure {
            echo "No se pudo eliminar el usuario ${USERNAME} debido a un error."
        }
    }
}

```
#### Paso 4 Ejecutar el Job

Darle click al botón play que dice "Build with Parameters". Se solicitará el parámetro de ```username```. Una vez completado, el pipeline eliminará el usuario y su directorio ```home```

#### Paso 5: Verificación de eliminación del usuario

Comprobar que se haya eliminado el usuario en la terminal de Linux
* Con ***Finger***:
```bash
finger nombreusuario

// Resultado esperado:

      finger: nombreusuario: no such user.

```
* Revisando el archivo ```/etc/passwd``` debe aparece el usuario que se acaba de crear:
```bash
cat /etc/passwd | grep nombreusuario

// Resultado esperado:

        (no debería mostrar nada)
```
Comprobar que se haya eliminado el directorio home del usuario en la terminal de Linux
* Revisando el listado de directorios de ```/home```:
```bash
ls -alh /home

// Resultado esperado:

        drwxr-xr-x  5 root          root         4.0K Nov 17 00:25 .
        drwxr-xr-x 22 root          root         4.0K Nov 16 10:48 ..
        drwxr-x---  9 ubuntu        ubuntu       4.0K Nov 14 17:48 ubuntu
```
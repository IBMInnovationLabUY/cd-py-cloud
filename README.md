# Cloud &amp; Data Paraguay - Journey to Cloud

En este repositorio encontrará los pasos y la guía para la parte práctica del workshop "Journey to Cloud"

## Introducción

Durante el transcurro de este hands-on nos basaremos en una aplicación para registrar jugadores de fútbol, la cual en principio se ejecuta de forma local junto a una base de datos MySQL. El código de esta aplicación lo puede encontrar en la carpeta *assets/code*. Lo que haremos sera crear una imagen de Docker de nuestra aplicación para poder despegarla en un cluster de Kubernetes, en el cual también desplegaremos la base de datos. Realizaremos las configuraciones necesarias para poder exponer de forma apropiada nuestra aplicación y configuraremos una toolchain en IBM Cloud para integrar DevOps. 
En forma demostrativa, se mostrara como realizar este procedimiento utilizando un cluster de Openshift y veremos cuales son sus ventajas por sobre Kubernetes.

## Prerrequisitos 


### 1 Crear un cluster en IBM Cloud


Es necesario que tengamos un cluster de Kubernetes en nuestra cuenta de IBM Cloud. Este hands-on está armado para que puedan seguirse todos los pasos usando un cluster free. 

*  Si tiene una cuenta de IBM Cloud, inicie sesión y luego haga clic [aqui](https://cloud.ibm.com/kubernetes/catalog/cluster/create) para crear el servicio.

*  Si aún no tiene cuenta de IBM Cloud, haga clic [aqui](https://ibm.biz/Bd26aa) para crear una. 

### 2 Instalar CLI de IBM Cloud  y Docker

Para poder subir nuestras imágenes a Container Registry y para poder operar con nuestro cluster de kubernetes, es necesario contar con la CLI de IBM Cloud. Haga clic [aqui](https://cloud.ibm.com/docs/cli?topic=cloud-cli-getting-started) para acceder a la documentación oficial de descarga. 

Al instalar la CLI también se debería instalar **automáticamente** docker. En caso de que esto **NO** suceda:

Para instalar docker le recomendamos que siga las instrucciones de su página oficial, puede acceder a ella haciendo clic [aquí](https://docs.docker.com/install/). Verá que en el menú lateral izquierdo están todas las plataformas en las que está disponible, solo debe hacer clic en la que usted esté usando y seguir los pasos allí indicados. 

### 3 Visual Studio Code

Puede descargarlo fácilmente desde la [web oficial](https://code.visualstudio.com/)

### 4 Descargar este repositorio

Al instalar la CLI de IBM Cloud, también se le instalará GIT por lo tanto podemos descargar este repositorio ejecutando el siguiente comando:


```
$ git clone https://github.com/IBMInnovationLabUY/cd-py-cloud.git
```


## Comencemos!

Si ya cumplimos con los prerrequisitos podemos comenzar! Lo que haremos ahora será abrir en el visual studio  la carpeta *assets*

Primero debemos configurar nuestro equipo para que trabaje con nuestro cluster en IBM Cloud. Para lograrlo debemos ingresar a la terminal en vs code  y loguearnos con nuestras credenciales.

```
$ ibmcloud login 
```

Ahora ejecutamos el siguiente comando en el cual deben reemplaza NOMBRE-DEL-CLUSTER, por el nombre del cluster que creamos. Esto nos descarga un archivo con las credenciales de nuestro cluster y ademas nos dara la ruta para configurar la variable KUBECONFIG. Debemos copiarlo y pegarlo en nuestra terminal. 


```
$ ibmcloud ks cluster-config --cluster NOMBRE-DEL-CLUSTER 
```

A continuación tendremos que setear la variable de entorno, copiando el EXPORT que nos aparece en consola.

```
export KUBECONFIG=/RUTA-DEL-ARCHIVO/kube-config-hou02-NOMBRE-DEL-CLUSTER.yml


NOTA: Este es un mero ejemplo, para que funcione deberá OBLIGATORIAMENTE copiar el EXPORT que le aparece después de la ejecución del anterior comando.

```

## Desplegar una base de datos MySQL en Kubernetes




Ahora que estamos listos, vamos a desplegar en nuestro Cluster de Kubernetes una base de datos MySQL. Para esto haremos uso del archivo mysql-deployment.yaml ubicado en la carpeta assets. 


Antes de hacer el despliegue, detengámonos a ver que tiene el archivo que vamos a utilizar:


```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
          # Use secret in real usage
        - name: MYSQL_ROOT_PASSWORD
          value: password
        ports:
        - containerPort: 3306
          name: mysql
```

Ejecutemos el siguiente comando para correr el deployment:

```
$ kubectl apply -f mysql-deployment.yaml

deployment.apps/mysql created

```
Veamos los pods de nuestro nodo para ver el estado de nuestro despliegue:

```
$ kubectl get pods

NAME                     READY   STATUS    RESTARTS   AGE
mysql-549454fd57-6sf5l   1/1     Running   0          55s
```

Por último vamos a exponer nuestro despliegue para poder acceder a MySQL y crear la bd para nuestra app. 

```
$ kubectl expose pod <NOMBRE-DEL-POD> --type=NodePort --name=servicio-mysql

service/mi-servicio-mysql exposed
```
Veamos la lista de servicios de nuestro cluster

```
$ kubectl get services
NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes          ClusterIP   172.21.0.1      <none>        443/TCP          9h
mi-servicio-mysql   NodePort    172.21.36.159   <none>        3306:31939/TCP   55s
```
Podemos ver que nuestra base de datos ya está expuesta en el **puerto 31939** de nuestro cluster. 
*Anoten el puerto de su servicio*

### Crear la base de datos

Ahora vamos a crear la base de datos y todas las tablas que necesitamos para nuestra app. 

Lo primero es conectarse por medio de bash al despliegue MySQL que realizamos en el paso anterior, para esto ejecutamos:

```bash
$ kubectl exec -it <NOMBRE-DEL-POD> bash

root@mysql-549454fd57-6sf5l:/# 
```
Ahora que estamos dentro de nuestro pod, ejecutemos mysql 

```bash
$ mysql -u root -p 
```
Recordemos que nuestra password es *password* tal como se indica en el archivo de nuestro deployment. 

Una vez hayamos ejecutado mysql ya podemos crear la base de datos que nuestra app necesita.


```mysql
$ CREATE DATABASE plantel;

Query OK, 1 row affected (0.00 sec)
```

Ahora creemos nuestra tabla para guardar los datos de los jugadores

 ```bash
 $ use plantel
 Database changed
 ```
 
**Ejecutar el comando en el archivo mysql.txt**

Listo! Ya está pronta nuestra base de datos en nuestro cluster de kubernetes!

 Nota:
 ```
 Use ctrl + D en la terminal para salir de mysql
 Vuelva a ejecutar ctrl + D para salir del pod
 ```

## Actualizar el código de la App

Antes de desplegar nuestra aplicación en kubernetes es necesario que modifiquemos el código de nuestra app para actualizar las credenciales de la base de datos. 
Ya tenemos nuestra base de datos desplegada, sabemos el puerto, el usuario y la password, solo nos hace falta conocer la IP pública de nuestro nodo. Hay varias formas de obtenerla, una forma es consultar por la descripción completa de nuestro nodo, para esto primero debemos saber el nombre de nuestro nodo. 

```bash
$ kubectl get nodes
NAME          STATUS   ROLES    AGE   VERSION
10.47.87.23   Ready    <none>   9h    v1.13.7+IKS
```
Como podemos ver, en mi caso el nombre de mi único nodo trabajador es 10.47.87.23

Ejecutemos el siguiente comando para obtener la descripción de un nodo:

```bash
$ kubectl describe node <NOMBRE-DEL-NODO>
```

Toda la información que se nos va a brindar es interesante y recomendamos que le de un vistazo. Para el propósito de este dojo solo nos va a interesar la siguiente sección donde se muestra la IP de del nodo:

```yaml
Addresses:
  InternalIP:  10.47.87.23
  ExternalIP:  184.172.233.164
  Hostname:    10.47.87.23
```

En mi caso la dirección IP del nodo es 184.172.233.164

Ahora actualicemos las credenciales de la base de datos, vayamos a la carpeta assets y lugo al archivo app.js, alli edifiquemos la *bd*, ingresen alli la dirección IP externa de su nodo, el puerto es el del servicio que expusimos, el usuario de la base de datos es *root* y la password que definimos en el .yaml era *password*.  por ejemplo en mi caso:


```js
const db = mysql.createConnection ({
    host: '184.172.233.164',
    port: '31939',
    user: 'root',
    password: 'password',
    database: 'equipo'
});
```

## Generar la imagen de la App

Ahora que tenemos nuestro codigo pronto, generemos una imagen de docker para desplegar en kubernetes. Volvamos a nuestra terminal en Visual e ingresemos a la carpeta *code*

```
$ cd code
```

Lo primero es armar el dockerfile

```bash
$ touch dockerfile
```
Con el comando de arriba generamos un archivo vacío del tipo dockerfile. Es importante que el mismo esté en la carpeta raíz de nuestro código.

Ahora pegamos lo siguiente dentro de nuestro dockerfile:

```dockerfile
FROM node:10
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 8080
CMD [ "npm", "start" ]
```

Ahora generemos el dockerignore

```bash
$ touch .dockerignore
```

Y  agregamos lo siguiente:

```dockerfile
node_modules
npm-debug.log
```
Una vez tengamos el dockerfile y el dockerignore podemos proceder a generar nuestra imagen:

```bash
$ docker build -t miapp:v1 . 
```

Podemos ejecutar el siguiente comando para listar todas las imágenes en nuestro equipo y ver la de nuestra app

```
$ docker images
REPOSITORY                                TAG                 IMAGE ID            CREATED              SIZE
miapp                                     v1                  77b551075347        About a minute ago   930MB
```

## Subir nuestra imagen a Container Registry (IKS)

Ahora subamos nuestra imagen a un repositorio privado. Usaremos un servicio de IBM Cloud llamado Container Registry que nos permite subir nuestras imágenes para almacenarlas y consumirlas en nuestros despliegues en Kubernetes. Ademas, entre otras cosas, realiza un escaneo de seguridad de nuestras imágenes.

Lo primero es logearse

```
$ ibmcloud cr login
```

Ahora creemos un namespace donde trabajar

```
$ ibmcloud cr namespace-add <NOMBRE_DEL_NAMESPACE>
```
Una vez creado el namespace,  hagamos un tag de nuestra imagen indicando el namespace y el repositorio, sino tenemos este creado, se va a crear automaticamente.

```
$ docker tag miapp:v1  us.icr.io/<my_namespace>/<my_repository>:<my_tag>
```

Por último debemos hacer un push de la imagen

```
$ docker push us.icr.io/<my_namespace>/<my_repository>:<my_tag>
```

Si queremos verificar, podemos utilizar el siguiente comando para ver el listado de nuestras imágenes almacenadas:

```
$ ibmcloud cr image-list
Listing images...

REPOSITORY                            TAG      DIGEST         NAMESPACE   CREATED        SIZE       
us.icr.io/code-day/panizza-code-day   latest   c3f99e8d11ca   code-day    18 hours ago   361 MB

OK
```


## Desplegar la App en K8s

Ahora que tenemos nuestra imagen en el repositorio, vamos a utilizarla para desplegar nuestra app en nuestro cluster de Kubernetes. 

Usaremos el archivo *app-deployment.yaml* que se encuentra en la carpeta *assets*. Veamos como esta formado:


```yaml
 apiVersion: v1
 kind: Deployment
 metadata:
   name: miapp
 spec:
   replicas: 3
   template:
     metadata:
       labels:
         app: web
     spec:
       containers:
         - name: front-end
           image: us.icr.io/<my_namespace>/<my_repository>
```

**Es importante que indiquemos en el archivo nuestra imagen**. Ahora, ejecutemos el despliegue:

Primero vayamos a la carpeta *assets*, si estamos en la carpeta *code* debemos ejecutar:

```
$ cd ..
```

Luego:
  
```
$ kubectl apply -f app-deployment.yaml
deployment.extensions/miapp created
```

Por último expongamos nuestro deployment para poder acceder a él utilizando NodePort:

```
$ kubectl expose deployment miapp --type=NodePort --name=expose-app --port=5000
```

Listo! Ya podemos ingresar a nuestra app, para acceder debemos averiguar qué puerto se le asignó a nuestro servicio *expose-app*

```
$ kubectl get services 
NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
expose-app       NodePort    172.21.52.5      <none>        5000:30983/TCP   23s
kubernetes       ClusterIP   172.21.0.1       <none>        443/TCP          16d
servicio-mysql   NodePort    172.21.32.104    <none>        3306:31460/TCP   6h9m
```

En mi caso para ingresar debo acceder a http://184.172.233.164:30983/










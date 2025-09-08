<h1>Dojo kubernetes, helm y argo</h1>

Contenido:
- [kubectl](#kubectl)
    - [descargar](#descargar)
    - [instalar](#instalar)
- [obtener credenciales de aws](#obtener-credenciales-de-aws)
- [configurar repositorio](#configurar-repositorio)
- [namespaces](#namespaces)
    - [configuración imperativa](#configuración-imperativa)
    - [configuración descriptiva](#configuración-descriptiva)
    - [realizar cambios en recursos](#realizar-cambios-en-recursos)
- [pods](#pods)
    - [consultar eventos y logs](#consultar-eventos-y-logs)
    - [crear nuestros pods](#crear-nuestros-pods)
- [replicaset](#replicaset)

# kubectl

<https://kubernetes.io/docs/tasks/tools/#kubectl>

## descargar
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" 

## instalar
    sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

validar:

    kubectl version --client

recomendación:

Agregar el alias en la terminal y/o en ~/.bashrc para todas las terminales

    alias k="kubectl"
    echo "alias k=\"kubectl\"" >> ~/.bashrc

# obtener credenciales de aws

Cuenta POC = DevOpseIngenieriaSBX

<https://d-906705dbfe.awsapps.com/start/#/?tab=accounts>

Recomendación: agregarlo en el archivo de credenciales de aws, para que la cli funcione en todas las terminales:

    vim ~/.aws/credentials

asi: 

    [default]
    aws_access_key_id=zzzzz
    aws_secret_access_key=zzzzz
    aws_session_token=zzzz==

Nota: Pegar las credenciales en el bloque [default], quitando la linea con el perfil [#####_BCO-PocRole]


validar la conexion aws:

    aws eks list-clusters --region us-east-2

Nota: En caso de error 509 (certificado), agregar la ruta del certificado en el archivo de configuración de aws:

    vim ~/.aws/config

asi: 

    [default]
    region = us-east-2
    ca_bundle = /etc/ssl/certs/bancolombia_certs.pem

configurar kubeconfig (kubectl)

    aws eks update-kubeconfig --region us-east-2 --name bcol-dev-vulkan-cluster

validar conexion con la api de kubernetes (eks)

    kubectl get pods

# configurar repositorio

Crear un repositorio **público**, puede ser en github.

    git init 
    git remote add origin <<repo-url>>

o 

    git clone <<repo-url>>

Crear la siguiente estructura de directorios:

```bash
mkdir k8s       # manifiestos
mkdir charts    # helm
```

# namespaces

Exploremos el cluster:

Listar namespaces existentes en el cluster:

    kubectl get namespaces

¿cuáles están configurados por defecto?

¿qué hace el siguiente comando?

    kubectl describe namespace default

Los tipos de recursos en kubernetes también usan nombres cortos, en este caso namespace = ns

Utilicemos los alias y el nombre corto de recurso para simplificar el comando:

    k get ns

Ahora, una vez identificados los namespaces, podemos preguntar por los recursos existentes dentro de cada uno, por ejemplo:

    kubectl get pods --namespace kube-system


## configuración imperativa

Implementar cambios directamente en el cluster

Creemos nuestro propio namespace, utilizando como prefijo  "dojo-", para diferenciarlo de los demás que ya existen en el sistema.

Podemos usar, por ejemplo, el nombre de usuario:

    echo "dojo-$(whoami)"

En mi caso, nombre del namespace sería:

```yaml
    name: dojo-johescob
```

Reemplazar en adelante **`<<my-namespace>>`** por el namespace que eligió en cualquier configuración de ahora en adelante.

Crear el namespace de forma imperativa:

    kubectl create namespace <<my-namespace>>

Obtener información del namespace

    kubectl describe namespace <<my-namespace>>

Y para obtener un detalle del recurso en formato YAML:

    kubectl get namespace <<my-namespace>> --output yaml

Y en JSON:

    kubectl get namespace <<my-namespace>> --output jsonn

Truco: si desea obtener solo una sección o un parámetro de la configuración, puede utilizar jsonpath, así:

    k get ns <<my-namespace>> -o jsonpath={.metadata.name}

## configuración descriptiva

A partir de este punto vamos a trabajar en el directorio `./k8s/`

    cd k8s

Para crear nuevamente el namespace, primero vamos a borrarlo

    kubectl delete namespace <<my-namespace>>

creamos el archivo `my-namespace.yaml`

    vim my-namespace.yaml

Contenido del archivo:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <<my-namespace>>
```

Realizamos la configuración declarativa indicandole a kubernetes que aplique los cambios en el archivo.  Como el recurso no existe, lo va a crear.

    kubectl apply -f my-namespace.yaml

No siempre es necesario conocer y crear el archivo desde cero, existe una forma de solicitarle a kubectl que emule una ejecución imperativa (`--dry-run`) y que muestre el manifiesto equivalente para una ejecución declarativa (`--output yaml`)

    kubectl create namespace <<my-namespace>> --dry-run=client --output yaml

Y si queremos llevarlo a archivo para luego aplicarlo, redireccionamos la salida a un archivo:

    kubectl create namespace <<my-namespace>> --dry-run=client --output yaml | tee -a my-namespace-dry.yaml

Para más información sobre las configuraciones imperativas o declarativas, puede consultar la documentación [Manage Kubernetes Objects](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/)

## realizar cambios en recursos

### vía declarativa: hacer los cambios en el manifiesto

Esta es la forma de trabajo recomendada, para mantener homologación, gestionar ciclo de vida (a través de git), y como seguro en caso de daño del cluster o de su base de datos (generalmente [etcd](https://etcd.io/)) 

Vamos a agregar el objeto `label` y vamos a crear una `clave: valor` con el código de la aplicación:

    vim my-namespace.yaml

```yaml
…
metadata:
  name: <<my-namespace>>
  labels:
    app-code: nu1234567
```

Aplicamos nuevamente la configuración.  A través del comando `apply` El sistema identifica los cambios (crear, actualizar, eliminar) y los implementa.  Podemos observar en la salida del comando cuáles acciones realiza sobre cada recurso definido.

    kubectl apply -f my-namespace.yaml


### vía imperativa: directamente en el cluster (runtime)

El comando `edit` abre una aplicación de edición (tipo `vim`), salvo que los cambios no se guardan en local sino **directamente** en la base de datos de kubernetes.  

    kubectl edit namespace <<my-namespace>>

Vamos a agregar una nueva etiqueta: (kubernetes.io...)

```yaml
…
metadata:
  labels:
    app-code: nu1234567
    kubernetes.io/metadata.name: <<my-namespace>>
```

Recorderis rápido de `vim`:

    Escribir:           [Esc][i]
    Salir de INSERT:    [Esc]
    Guardar y salir:    [Esc][:][w][q][Enter]
    Salir sin guardar:  [Esc][q][!][Enter]

Enlace de interés: [vim cheatsheet](https://devhints.io/vim)

Nota: En algunas ocasiones, si los cambios solicitados no se pueden ejecutar, el sistema guarda los cambios en un archivo local temporal sin aplicarlos en el cluster, por ejemplo:

    A copy of your changes has been stored to "/tmp/kubectl-edit-2045177202.yaml"
    error: ...

Finalmente, podemos borrar el recurso de forma declarativa (referenciando el mismo archivo) o imperativa, así:

    kubectl delete -f my-namespace.yaml         # declarativo

o

    kubectl delete namespace <<my-namespace>>   # imperativo

Vamos a deshacer esa eliminación, volvamos a crear el namespace, porque lo vamos a necesitar en adelante.

    kubectl apply -f my-namespace.yaml

Observemos el detalle del namespace:

    k describe ns <<my-namespace>>

¿donde está la última etiqueta aplicada?

<details>
<summary> Ver respuesta </summary>
  Ese cambio se perdió para siempre (en teoría *),  debido a que fue una modificación directamente en el cluster.

  Así, que esta vez: ¡haz el cambio declarativo, en el archivo, y aplicalo nuevamente!
  
  \* En teoría, porque es posible hacer copias de seguridad y restauraciones en etcd, pero es tanto camello que para este caso no vale la pena, fue una modificación muy pequeña.

</details><br>

# pods

Explorar qué encuentro en el cluster

    kubectl get pods --namespace kube-system

Observar qué información obtengo con el siguiente cómando:

    kubectl get pods –A

Efectivamente, cuando no conozco el namespace en el que está desplegado un recurso, puedo utilizar el parámetro -A, que significa: **All namespaces**

Ejercicio: Para el namespace `kube-system`, ¿cómo identifico en que nodo esta corriendo cada pod?

Pista: [List the pods](https://kubernetes.io/docs/tasks/access-application-cluster/service-access-application-cluster/)

<details>
<summary> Ver respuesta </summary>
  Hemos utilizado el parámetro -o o --output para presentar la salida en diferentes formatos, pero también es posible utilizar el valor <code>wide</code> para obtener mayor detalle. En el caso de los pods, muestra el nodo en el que está corriendo cada pod, así:

  <code>kubectl get pods --namespace kube-system -o wide</code>

</details><br>

## consultar eventos y logs

Al describir cualquier recurso en kubernetes, se presenta al final el listado de los últimos eventos asociados a dicho recurso.

Vamos a utilizar alguno de los pods del namespace `kube-system` identificados en el paso anterior

    kubectl describe pod <<aws-node-#####>> --namespace kube-system

Analice la salida del comando. ¿Qué información obtienen?

Otra opción es consultar únicamente este bloque de información con el comando `events`:

    kubectl events <<aws-node-#####>> --namespace kube-system

Adicionalmente, se pueden obtener los logs de un POD con el siguiente comando:

    kubectl logs <<aws-node-#####>> --namespace kube-system

Recordemos que un POD se compone de uno o varios contenedores.  Según esto, ¿que observa en el comando anterior? ¿se presenta información de todos los contenedores o de uno en particular?

<details>
<summary> Ver respuesta </summary>
  Kubernetes presenta información de un solo contenedor, identificado como <code>default contaner</code>. Para ver los logs de otro contenedor, puede utilizar el parámetro: <code>-c  &lt;&lt;container-name&gt;&gt;</code>, por ejemplo:
  <br><br>
  <code>kubectl logs &lt;&lt;aws-node-#####	&gt;&gt; -c aws-eks-nodeagent --namespace kube-system</code>

</details><br>

Ejercicio: Mostrar los nombres de los contenedores del POD `<<aws-node-####>>` con jsonpath

<details>
<summary> Ver respuesta </summary>
  Si obtienen el detalle del POD con <code>describe</code>, encontrarán una salida muy extensa, igual que con <code>get -o yaml</code> o <code>get -o json</code>.  En estos casos es muy util filtrar bloques o atributos con jsonpath, así: <br><br>
  <code>k get pod &lt;&lt;aws-node-#####&gt;&gt; -n kube-system -o jsonpath={.spec.containers[*].name} </code>
</details><br>


## crear nuestros pods

Vamos a crear dos instancias de NGINX (2 PODs) en nuestro *namespace*, el primero de forma imperativa y el segundo declarativa.

**imperativo:**

    kubectl run my-nginx-pod-1 --image=nginx --labels="app-code=nu1234567" --namespace <<my-namespace>>

El primer parámetro `my-nginx-pod-1` corresponde al nombre que se le dará al POD

verificar:

    kubectl get pods -n <<my-namespace>>

**declarativo:**

Creamos el archivo `my-pod-2.yaml`

    vim my-pod-2.yaml

Comencemos por los 3 bloques que conocemos, esta vez `kind: Pod`, el resto de claves ya son conocidas.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx-pod-2
  namespace: <<my-namespace>>
  labels:
    app-code: nu1234567
```

Y agregamos el bloque `spec:` con las especificaciones técnicas del POD y sus contenedores:

```yaml
...
spec:
  containers:
  - image: nginx
    name: my-nginx-container
```

Observe que la clave `containers` es una lista `(-)`, de esta manera puede especificar uno o más contenedores.

Guardamos los cambios, aplicamos y verificamos:

    kubectl apply -f my-pod-2.yaml
    kubectl get pods -n <<my-namespace>>

Para finalizar, vamos a desplegar otra aplicación en nuestro namespace

    kubectl run monitoreo --image=dynatrace --namespace=<<my-namespace>>
    kubectl get pods -n <<my-namespace>>

¿qué pasa con el nuevo POD?

<details>
<summary> Ver respuesta </summary>
  Con el comando <code>get pods</code> puede observar que el estado (STATUS) del POD es "ErrImagePull".<br><br>
  Adicionalmente, ejecutando <code>kubectl events monitoreo -n &lt;&lt;my-namespace&gt;&gt;</code> puede encontrar el detalle de cada paso realizado por el sistema para crear el pod: Scheduled, Pulling, Failed"<br><br>
  En este caso podemos observar que la imagen no la encuentra en el repositorio público de docker, lo cual es correcto. Para crear los PODs correctamente, se requiere una referencia exacta de la ubicación de la imágen a utilizar, no todas se encuentran en docker.io.
</details><br>

A continuación vamos a eliminar el POD `monitoreo` de nuestro *namespace*:

Intentemos eliminarlo de forma declarativa.
<details>
<summary> Ver respuesta </summary>
  ¿Dónde está el archivo o manifiesto con el que fue creado? <br><br> Si observas nuevamente, el POD fue creado de forma imperativa, por lo tanto no hay un archivo declarativo que se pueda usar para la eliminación.<br><br>

  En este caso, la solución es eliminarlo de la misma manera: de forma imperativa:<br><br> 

  <code>kubectl delete pod monitoreo -n &lt;&lt;my-namespace&gt;&gt;</code> 
</details><br>

## replicaset

En este momento debemos tener 2 PODs de NGINX en estado Running, uno creado de forma imperativa y el otro declarativa.

    kubectl get pods -n <<my-namespace>>

    NAME             READY   STATUS    RESTARTS   AGE
    my-nginx-pod-1   1/1     Running   0          #m##s
    my-nginx-pod-2   1/1     Running   0          #m##s

Antes del siguiente paso, donde vamos a homologar la configuración de `my-nginx-pod-1` para que sea declarativa, es util conocer que un archivo declarativo puede contener uno o varios recursos de kubernetes, es decir, varios bloques `apiVersion + kind + metadata + spec`.  Todo lo que se necesita es separar cada bloque con una línea con 3 guiones `---`, así:

```yaml
#recurso 1
apiVersion: ...
kind: ...
---
#recurso n
apiVersion: ...
kind: ...
```

Ejericio: Homologar en el archivo my-pod-2.yaml los 2 PODs.

Opción 1: Editar el archivo y agregar el segundo bloque manualmente.  PD: Recuerde cambiar el nombre del POD a `my-nginx-pod-1`

Opción 2: Obtener la configuración declarativa equivalente y anexarla al final del archivo. PD: No olviden eliminar el bloque no declarativo `status`

<details>
<summary> Ver respuesta </summary>
<code>echo "---" >> my-pod-2.yaml<br>
k get pod my-nginx-pod-1 -o yaml -n &lt;&lt;my-namespace&gt;&gt; >> my-pod-2.yaml</code><br><br>
Como podrá observar, kubernetes se encarga de configurar claves que utiliza por defecto a la hora de desplegar los recursos, como por ejemplo los tipos de nodo a utilizar (taints and tolerations), volumenes con información de certificados, etc. Puede dejar estas claves en el archivo de configuración o simplemnete eliminarlas, igual van a ser configuradas nuevamente cuando se aplique el manifiesto.
</details><br>

Pregunta: ¿Los contenedores pueden tener el mismo nombre `my-nginx-container` o también debemos agregarle un sufijo para aplicar el manifiesto? ¿porqué?

Antes de aplicar, debemos eliminar el POD creado de forma imperativa  `my-nginx-pod-1`

    k delete pod my-nginx-pod-1 -n <<my-namespace>>

    k apply -f my-pod-2.yaml

    pod/my-nginx-pod-2 unchanged
    pod/my-nginx-pod-1 created


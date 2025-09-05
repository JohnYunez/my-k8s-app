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
    app-code: nu0123456
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
    app-code: nu0445001
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
  
  \* En teoría, porque es posible hacer copias de seguridad y restauraciones en etcd, pero es tanto camello que para este caso no vale la pena, debido a que fue una modificación directamente

</details>


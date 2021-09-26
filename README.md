# GitOps - Continuous Delivery with ArgoCD

![ArgoCD](images/argocd.png)
Lo que vamos a ver en este repo es el ejercicio práctico de los conceptos de GitOps en lo que respecta a CD. 

1. Introducción
2. Que es ArgoCD?
3. Conceptos básicos 
4. Instalación en Minikube 
5. Caso de Usos (Despliegues).
    - 1. Despliegue utilizando la linea de comando (CLI) de ArgoCD
    - 2. Despliegue en producción basado en principios de GitOps (ambiente estáticos)
    - 3. Despilegue en desarrollo basado en principios de GitOps (ambiente estáticos de prueba)
    - 4. Despliegue basados en un proceso de Pull o Merge Request (ambientes efímeros)

## 1. Introducción

El despliegue de aplicaciones en la via tradicional es realizado manualmente o de manera automática con una herramienta de despliegue e integración continua como Jenkins. Donde un operador, o eventualmente si un plugin está implentado, por medio de un Pull Request (PR). El sistema de integración continua es el encargado de crear el artefacto y eventualemnte desplegar el artefacto en la infraestructura destino realizando un "Push" del nuevo release.

GitOps es una alternativa al paradigma tradicional donde contamos con un mecanismo que detecta los cambios en nuestro repositorio Git - pull del estado deseado - renderiza los manifiestos y los aplica en el cluster de kubernetes destino. Todo esto sin intervención de un Operador y sin tener realizar un push sobre la api externa de Kubernetes.

Para mas detalle en las estrategias de Push y Pull ingresar al link: [Push vs Pull](https://www.weave.works/blog/gitops-operations-by-pull-request). 

Existen varias herramientas de GitOps en la comunidad que nos ayuden a detectar los cambios y aplicarlos en uno o mas clusters. Un analisis de ArgoCD, FluxCD y Jenkin-X pueden encontrarlo [acá](https://docs.google.com/document/d/10_3mESUD85cz3yRskLyrtSLR0v0LhRUSqGj1MyTNm_w). En nuestro caso hemos seleccionado [ArgoCD](https://argoproj.github.io/) y lo explicamos en el link adjunto. 

## 2. Que es ArgoCD?

En el sitio del proyecto encontramos la definición escrita por la propia comunidad de ArgoCD y dice "**Argo CD is a declarative, GitOps continuos delivery tool for Kubernetes"**. En definitiva, es una herramienta que nos ayuda a desplegar aplicaciones de manera fácil en Kubernetes. El deploy de las aplicaciones y la administración del ciclo de vida será automatizado, auditable, controlado por versión y facil de entender (escrito en manifiestos).

Argo Project es un conjunto amplio de herramientas, entre ellas podemos encontrar ArgoCD, Argo Rollouts, Argo Workflow, Argo Events y Argo Pipelines. En nuestro caso, solamente nos enfocaremos en ArgoCD como herramienta de Continuous Delivery.

## 3. Conceptos Básicos

ArgoCD tiene posee un glosario de termino ámplio que es necesario tener presente a la hora de trabajar con GitOps y ArgoCD. Dejamos un diagrama de la arquitectura y un abreve explicación de todos los componentes que intervienen.

![ArgoCD](images/argocd_architecture.png)

- *[ArgoCD (Tools)](https://argoproj.github.io/argo-cd/operator-manual/architecture/)*: Herramienta declarativa para el despliegue de aplicaciones en Kubernetes basadas en los principios de GitOps.
  - *API Server*: Expone la API (gRPC/REST) que consume la Web UI, CLI y CI/CD tools. Administración de aplicaciones y reporte de estado.
  - *Repository Server*: Servicio interno que manitiene localmente la cache of del repositorio Git y responsable de generar los manifiestos de kubernetes renderizados.
  - *Application Controller*: Servicio que monitorea continuamente el estado de las aplicaciones y la compara con el estado actual definido en git.
- *[Declarative Setup](https://argoproj.github.io/argo-cd/operator-manual/declarative-setup/)*
  - *Application (Application)*: Es el recurso en kubernetes definido en un CRD, que representa la aplicación desplegada en un ambiente. Define:
    - `source` el repo de la definición de la aplicación.
    - `destination` el cluster donde se desplegará. Permite desplegar apps de apps, esto es controlarel despliegue de apps definiendolas en otra app (ver ejercicio)
  - *Projects (AppProjects)*: Es el recurso de kubernetes definido en un CRD que representa la agrupación lógica de `applications`.
    - `sourceRepo`: referencia la lista de `applications` relacionadas con el proyecto.
    - `destinations`: referencia los `clusters` y `namespace` que las aplicaciones en el proyecto puede desplegar.
    - `roles`: lista de entidades a las que las aplicaciones pueden acceder. Se definen `whilelist` and `blacklist` de recursos de kubernete.
  - *[Repositories](https://argoproj.github.io/argo-cd/operator-manual/declarative-setup/)*: `Secrets` de kubernetes que son usados para definir de donde ArgoCD tomará los manifiestos. Es recomendable usar [sealed-secrets](https://github.com/bitnami-labs/sealed-secrets). En cada objeto `repositories` quedarán definido `url`, `username`, `password`, `sshPrivateKey` (SSH), `githubAppPrivateKey`.
  - *Clusters*: Las credenciales de los clusters son almacenadas en secrets igual que los repositorios de credenciales. En un objeto de tipo `cluster` definiremos `name`, `server`, `namespace` y `config` (json format).
- *ArgoCD States*
  - *Target State*: Estado deseado definido en el repositorio de código.
  - *Live State*: Estado de los pods deplegados.
  - *Sync Status*: Es el estado definido de la aplicación el mismo que el estado en vivo?.
  - *Sync*: Representa el proceso de hacer que una aplicación pase del estado deseado al estado en el repositorio git al estado en vivo (live state).
  - *Refresh*: Proceso que dispara la comprobación del estado deseado con el estado con el estado actual.
  - *Health*: El estado de salud de la aplicación.

## 4. Install ArgoCD in Minikube

Para la instalación de ArgoCD utilizamos [Helm](https://helm.sh) sobre [Minikube](https://minikube.sigs.k8s.io/docs/start/). Crearemos un conjunto de variables que personalizará la instalación de ArgoCD.

1. Clonamos el repo y exportamos variables.

```
export GIT_USER=<your_git_user_name>
git clone https://github.com/$GIT_USER/gitops-argocd.git
cd gitops-argocd
export INGRESS_HOST=(minikube ip)
```

2. Agregamos repo de helm y creamos variables para la instalación. 

```
helm repo add argo https://argoproj.github.io/argo-helm

cat << EOF >> argocd/argocd-values.yaml
server:
  ingress:
    enabled: true
  extraArgs:
  - --insecure
installCRDs: false
EOF
```

3. Creamos el namespace donde se alojará argo y donde se crearon los recursos propios de argo como `application` y `appprojects`

```
kubectl create namespace argocd
```

4. Realizamos al instalación de argocd

```
helm upgrade --install argocd argo/argo-cd --namespace argocd --version 2.8.0 --set server.ingress.hosts="{argocd.$INGRESS_HOST.nip.io}" --values argo/argocd-values.yaml --wait
```

  NOTA: Minikube utiliza el dominio `minikube ip`.nip.io como local, si por ejemplo la ip de nuestro minikube es 192.168.64.6 el dominio seráe argocd.192.168.64.6.nip.io.

El chart de helm desplegará bajo el namespace `argocd` los pods del server, application controller y repo controller.

```
➜  gitops-argocd k get pods -n argocd
NAME                                             READY   STATUS    RESTARTS   AGE
argocd-application-controller-84d4b5ccfd-52c5l   1/1     Running   0          2d19h
argocd-dex-server-66bc5cb6b-ck9mx                1/1     Running   0          2d18h
argocd-redis-64d7554df6-wfsgs                    1/1     Running   0          2d19h
argocd-repo-server-798cb4786b-dsrmj              1/1     Running   5          2d19h
argocd-server-77cd6587c7-dqlj8                   1/1     Running   0          2d19h
```

5. Login a la API de ArgoCD

Para interactuar desde la CLI como desde la Web, es necesario realizar un login. El usuario por defecto es `admin` y la password por defacto es el nombre del pod server `argocd-server-xxxxxx`.

Exportamos la password por default.

```
export PASS=$(kubectl --namespace argocd get pods --selector app.kubernetes.io/name=argocd-server --output name | cut -d'/' -f 2)
```

Realizamos el login a via CLI

```
argocd login --insecure --username admin --password $PASS --grpc-web argocd.$INGRESS_HOST.nip.io
```

Actualizamos la password por defecto por admin/<lo que quieran>

```
argocd account update-password
```

Testeamos la url desde el navegador.

```
open http://argocd.$INGRESS_HOST.nip.io
```

Teniendo los accesos, podemos desplegar nuestra primera aplicación con ArgoCD. Cabe destacar que todos las acciones que haremos desde la linea de comandos lo podremos hacer desde la consola web por medio de la URL [http://argocd.$INGRESS_HOST.nip.io.](http://argocd.$INGRESS_HOST.nip.io)

## 5. Casos de Uso
### 1. Despliegue utilizando la linea de comando (CLI) de ArgoCD 

Una vez instalado ArgoCD el primer paso será desplegar una aplicación desde la linea de comandods. Es fácil pensar que este proceder es lo mismo que poder hacerlo desde la linea de comandos de kubernetes `kubectl`, pero si nos centramos en los conceptos anteriores de `Projects` y `Applications` hay una pequeña diferencia en el alcance. Projects como Applications son recursos de kubernetes, como si lo fuesen los pods, services, etc. Estos recursos pueden ser creados desde la cli de argo `arcocd` como desde manifiestos que veremos en los escenarios 2, 3 y 4. 

Si el objetivo es desplegar una aplicación, es verdad que interfacear con la API de Argo o la API de kubernetes es lo mismo, la diferecnai puntual esta en la estructura y opciones de definir aplicaciones con Argo. En Argo vamos a declarar la fuente donde estarán alojados los manifiestos de nuestra aplicación (repo git), el namespace donde se creará la aplicación y además el o los clusters de kubernetes donde desplegaremos la aplicación. En este ultimo punto es donde está la diferencia, podemos alcanzar uno o mas cluster de kubernetes utilizando la cli de argo entre otras opciones mas, como por ejemplo monitorear el estado.

Ahora bien, para desplegar una aplicación necesitamos alojarla en un proyecto de argo - resource `AppProject` - y en caso de no indicarse uno especifico - al igual que los namespaces de kubernetes - se aloja en el `AppProject` `default`. En lo que respecta a cluster de kubernetes, si no tenemos definido ninguno adicional la aplicación se va a instalar en el cluster donde se encuentran desplegados ArgoCD, en este caso será nuestro minikube con url `https://kubernetes.default.svc`.

Antes de desplegar la aplicación veamos los recursos de kubernetes AppProject y Application, creados por defecto en la instalación de ArgoCD.

Desde la linea de comandos de argo

```
argocd proj list
```

desde la linea de comandos de kubernetes

```
kubectl get appproject -n argocd
```

Ahora crearemos un namespace para poder alojar los recursos de aplicación.

```
kubectl create namespace voting-app
```

Teniendo disponible un proyecto `default`, le namepsace aplicativo `voting-app` y un cluster en donde aplicar los cambios `http://kuberneetes.default.svc`, creamos la aplicación.

```
argocd app create voting-app --repo https://github.com/$GIT_USER/voting-app.git --path "./helm" --dest-server https://kubernetes.default.svc --dest-namespace voting-app
```

Al ejecutar la instrucción previa ArgoCD creará los recuros `Application` bajo el namespace `argocd` en el projecto por default con nombre `default`.

Listamos los recursos de proyecto `voting-app` y veremos que no tenemos ningun objeto de kubernetes creado.

```
kubectl -n voting-app get all
```

No encontramos ningun recurso en el proyecto aplicativo porque no le indicamos a argo que haga un sync al momento de crear la aplicación. Para poder forzar un sync lo hacemos desde la linea de comandos de argo.

```
argocd app sync voting-app
```

Verificamos que todo esté sincronizado desde la cli de argo y desde la cli de kubernetes.

```
argocd app list voting-app
kubectl get all -n voting-app
```

Por ultimo podemos ver los objetos creados desde la UI de ArgoCD

```
open https://argocd.$INGRESS_HOST.nip.io
```

Eliminar una aplicación y todos los recursos de kubernetes asociados a ella es muy simple, solo tenemos que eleminar la definición de `Application` en ArgoCD y él mismo se encargará de eliminar todos los recursos asociados a la definición de la aplicación.

```
argocd app delete voting-app
```

ArgoCd no elimina le namespace que previamente creamos para alojar la aplicación, por lo que lo hacemos de manera manual.

```
kubectl delete ns voting-app
```

Como hemos visto, utilizar la CLI nos permite monitorizar desde nuestra terminal la evolución den nuestra app. Por otro lado la CLI nos ayuda a desplegar los manifiestos de ArgoCD (Application y Projects) de manera fácil. Asociado a esto todos los recursos de kubernetes asociados a nuestra aplicación sin la necesidad de interactuar con la API de Kubernetes. Es cierto, que si bien no se alinea con la definición que estamos manejando de GitOps, estas acciones pueden ser definidas en un pipeline de CI/CD en gitlab-ci, github-actions o jenkins pipeline.

## Deploy production with GipOps Principles

El siguiente paso de este ejercicio es poder desplegar la misma aplicación basada en un proceso puro de GitOps, esto quiere decir manejaremos manifiestos YAML definidos en un repositorio de código (gitlab, github, etc) y asociado el flujo de aprobación/promoción de código entre repos/branches, junto con el monitoreo de ArgoCD, nos permitirá desplegar aplicaciones en los cluster de kubernetes sin la necesidad de interactuar con la API de Kubernetes y tampoco con la API de ArgoCD. Veamos como puede ser el flujo de operación. 

### Repositorios

Antes de usar argo tenemos que tener en claro una estrategia de organización de repositorios que nos permitirá organizar nuestros despliegues. Vamos a contar con los siguientes repos:

- **Application**: [voting-app](https://github.com/gonzaloacosta/voting-app.git). Repositorio donde se realizarán las iteraciones de código por parte de los desarrolladores (developers).
- **Production**: [gitops-voting-app-production](https://github.com/gonzaloacosta/gitops-voting-app-production.git). Definición del ambiente productivo apunta a la rama master de voting-app. Repositorio donde se alojaran los manifiestos de de ArgoCD que haran referencia al repositorio de aplicación (voting-app) y al cluster donde estaremos desplegando la aplicación.
- **Development**: [gitops-voting-app-development](https://github.com/gonzaloacosta/gitops-voting-app-development.git). Definición del ambiente de desarrollo y apunta a la rama develop en voting-app.
- **Previews**: [gitops-voting-app-previews](https://github.com/gonzaloacosta/gitops-voting-app-previews.git). Definición del ambientes efímeros apuntan a las famas de feature branch de los desarrolladores.

### Ramas Aplicativas

En el repositorio de aplicación tendremos las siguientes ramas:

- master: Apunta apunta a los cambios del ambiente productivo y es el código estable.
- develop: El acumulado de iteraciones antes de sacar un nuevo release, todos los feature branch realizan el merge sobre develop para luego pasar a master
- pr-1 y pr-2: feature branch, nueva funcionalidad aplicativa que será mergeada previa aprobación del Pull Request creado por el desarrollador. Utilizada la rama para crear ambientes efímeros.
### Operación (GitFlow para ArgoCD [Propuesta])

Contamos con un repositorio de código como github o gitlab y formamos parte de un equipo de trabajo con los siguientes roles (genéricos):

- **Branches**: En un repositorio de applicativo tendremos una rama estable llamada rama <master o main>, una rama de desarrollo <develop> donde los desarrolladores realizar el merge de sus feature branchs y las ramas de <feature> donde crearán los cambios. No analizaremos ramas de <hotfix|bugfix> por cuestiones prácticas.
- **Developer**: Persona que realizará las modificaciones sobre el repositorio de código de aplicacion [voting-app](https://github.com/gonzaloacosta/voting-app). El developer realiza un Pull Request (PR) sobre la rama `develop` a partir de una rama de `feature`. No realiza PR sobre master.
- **Reviewer Dev**: Persona encargada de realizar la revisión del a PR del Developer que habilitará la acción de Merge contra la rama develop, esta PR habilitará la generación del artefacto - componente de software que será el driver de delivery -.
- **Operator**: Persona con el rol de DevOps / SRE (PROD) encargada de realizar PR sobre los repositorios de Operación [https://github.com/gonzaloacosta/gitops-voting-app-production).
- **Reviewer Ops**: Persona encargada de realizar la revisión del a PR del Operator, esta PR habilitará la generación de infraestructura en production.
## Git Flow

  > Developer crea un feature branch [PR-1] 
    > Reviewer Dev aprueba o rechaza el feature branch PR-1 [Accept-PR-1] 
      > Deploy Develop or Preview Environment si se acepta el Approbe y posterior Merge

  > Operator [PR-1] crea un branch para agregar un nuevo ambiente (por ejemplo asociado al PR1)
    > Reviewer Ops acepta o rechaza el branch PR-1 [Accept-PR-1] 
      > Deploy Prod si es aceptado el PR-1 y posteriormente Mergeado.

### Desplegar la aplicación con ArgoCD en un circuito de GitOps

El primer caso de GitOps con ArgoCD es el de un ambiente estático, esto quiere decir un ambiente que perdurá en el tiempo y ofrecerá servicio de manera constante a clientes, por ejemplo un ambiente productivo o un ambiente de staging. La idea siguiente es definir un recurso Application  
como manifiesto YAML que apunten al repo que contiene los manifiestos de despliegue de la aplicación de negocio `voting-app`. Eso es ArgoCD Application production despliegua el template helm de voting-app.

Creamos el manifiesto del recurso `AppProject` llamado `production` donde será alojado el recurso `Application` con nombre `production` que tendrá la definición de la aplicación `voting-app`. El manifiesto de `AppProject` define los repositorios a utilizar `sourceRepo`, el `namespace` y servidores de kubernetes `server` y además los recursos de Kubernetes que pueden utilizar las aplicaciones que se encuentren en este proyecto. Podemos agregar whitelist como blacklist de recursos.

Clonamos el repo que tendrá la definición de la aplicación y el proyecto de argo.

```
export GIT_USERNAME=<your pretty username>
git clone https://github.com/$GIT_USERNAME/gitops-voting-app-production.git
cd gitops-voting-app-production
```

Creamos el manifiesto del proyecto.

```
cat << EOF >> project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  description: Production project
  sourceRepos:
  - '*'
  destinations:
  - namespace: '*'
    server: https://kubernetes.default.svc
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
  namespaceResourceWhitelist:
  - group: '*'
    kind: '*'
EOF
```

Listamos desde la CLI y desde la console Web.

```
kubectl get appproject

open http://argocd.$INGRESS_HOST.nip.io/settings/projects
```

El concepto es intersante, tendremos un repo de operacion que tendrá un chart de heml donde se definiran los recursos de ArgoCD que representarán a los ambientes. Esto es utilizaremos un chart de helm para desplegar aplicaciónes en producción, en nuestro caso `voting-app`. Dentro del repo de `voting-app` tenemos el chart de helm que define como se despliega. 

  > Chart de Helm (Ops) despliega Chart de Helm (Apps)

```
mkdir -p helm/templates
cat << EOF >> helm/Chart.yaml
apiVersion: v1
description: Production environment
name: voting-app
version: "0.0.1"
EOF
```

Creamos dentro del template del chart el recurso de tipo `Application` de argo llamado `voting-app` que definirá el `source` (helm, repo, values) y `destination` (namespace y server). 

  > NOTA: Es util y necesario remarcar que es aquí donde podemos distribuir nuestra aplicación en una estrategia de despliegue multi-cluster debido a que una misma aplicación podemos aplicarla desde un único lugar en multiples clusters.

```
cat << EOF >> helm/templates/voting-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: voting-app
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: production
  source:
    path: helm
    repoURL: https://github.com/gonzaloacosta/voting-app.git
    targetRevision: HEAD
    helm:
      values: |
        result:
          image:
            tag: latest
          replicas: 2
          ingressHost: result.192.168.64.6.nip.io
        vote:
          image:
            tag: latest
          replicas: 2
          ingressHost: vote.192.168.64.6.nip.io
        worker:
          image:
            tag: latest
          replicas: 2
      version: v3
  destination:
    namespace: production
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
EOF
```

Por último nos queda definir el recurso de tipo `Application` pero que hace referencia al proyecto de productión de ArgoCD. Application Ops despliega Application Apps.

```
cat << EOF >> apps.yaml                                            
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: production
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: production
  source:
    repoURL: https://github.com/gonzaloacosta/gitops-voting-app-production.git
    targetRevision: HEAD
    path: helm
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
    - CreateNamespace=true
EOF
```

Solo resta aplicar el cambio (por unica vez) para que ArgoCD haga el resto, crear todos los recursos y sincronizar los repos.

```
kubectl -n argocd apply -f apps.yaml
```

Chequeamos desde la consola web el mapa de relación de recursos.

```
open http://argocd.$INGRESS_HOST.nip.io
```

Teniendo en cuenta solo por única vez se deben crear los recursos de ArgoCD. En las iteraciones posteriores vamos a poder crear y/o modifcar el ambiente sin tener que tocar la API de Kubernetes o la API de ArgoCD. Solo lo haremos utilizando un flujo puro de GitOps. 

### 3. Despilegue en desarrollo basado en principios de GitOps (ambiente estáticos de prueba)

El caso de uso es similar al primero siendo un ambiente estático, la diferencia se presenta en los branches a los que apunta el nuevo ambiente. En este caso utilizamos un nuevo repo de ambiente y este repo cambia las variables que toman como values el chart de helm.

Creamos el namespace donde se aloja la aplicación.

```
kubectl create ns development 
```

Construimos nuevos artefactos pero con el tag de develop.

```
cd voting-app
git checkout -b develop
cat helm/values.yaml | sed -e 'g@image: latest@image: develop@g
docker-compose build .
docker push gonzaloacosta/voting-app-worker:develop
docker push gonzaloacosta/voting-app-result:develop
docker push gonzaloacosta/voting-app-vote:develop
```

Creamos un nuevo repo llamado `gitops-voting-app-develop` que tendrá la misma estructura que production pero tendrá la defición de un proyecto llamado `development` y un recurso `Application` de Ops que utilizará el chart de helm en la rama develop para desplegar la aplicación con el nuevo cmabio. Particularmente en los values del chart de helm de aplicación (app) podremos definir diferentes requerimientos que prod, por ejemplo menor cantidad de pods por replicas, sin persistencia, etc.

Creamos el proyecto y aplicación.

```
cd ../gitops-voting-app-develop
kubectl apply -f project.yaml
kubectl apply -f apps.yaml
```

Verificamos que los recursos se hayan creado.

```
open http://vote.develop.$INGRESS_HOST.nip.io

open http://result.develop.$INGRESS_HOST.nip.io
```
## Create preview environment without touch kubernetes cluster


### PR-1

El último caso de uso tiene como objetivo poder crear ambientes de manera dinámica una vez aprobado el Pull Request donde es agregado un nuevo manifiesto dentro del directorio templates con la definición del nuevo ambiente a crear asociado al feature branch. Este escenario se dá cuando un developer necesita crear un ambiente par aprobar su código y una vez probado - cuando es realizado el merge - poder sacar ese archivo de los ambientes definidos para que ArgoCD lo elimine, esto nos permite poder crear ambiente bajo de manda.

Creamos un repositorio que tendrá la misma estructura que los dos pasados con la diferencia que a medida que se requirea se irán agregando ambientes a demanda.

```
git clone https://github.com/gonzaloacota/gitops-votin-app-previews.git
cd gitops-votin-app-previews
```

Creamos el proyecto donde se alojaran las aplicaciones.

```
cat project.yaml
kubectl apply -f project.yaml
```

Creamos nuevos artefactos con pequeños cambios para poder testear.

```
cat helm/values.yaml | sed -e 's@tag: latest@tag: pr-1@g' | helm/values.yaml
cat docker-compose.yml | sed -e 's@:develop@:pr-1@g' | tee docker-compose.yml
docker-compose up -d 
docker push gonzaloacosta/voting-app-worker:pr-1
docker push gonzaloacosta/voting-app-result:pr-1
docker push gonzaloacosta/voting-app-vote:pr-1
```

Preparamos un nuevo ambiente donde el nombro del namespace donde se alojarán está relacionado con el numero de la PR, ej PR-1. Cabe mencionar que aquí podemos utilizar cualquier variable de ambiente que sea relevante para nuestro flujo, por ejemplo el BUILD_ID, PIPELINE_ID, PR_ID, etc. 

Exportamos las variables de ambiente que utilizaremos en nuestro flujo de trabajo, si bien lo hacemos de manera manual puede utilizarse en un ciruito de ci/cd con gitlab-ci, github-action, jenkins, etc.

```
export PR_ID=1
export REPO=voting-app
export APP_ID=pr-$REPO-$PR_ID
export IMAGE_TAG=pr-1
export HOSTNAME=$APP_ID.$INGRESS_HOST.nip.io
```

Tomando un template como base, donde reemplazaremos las variables de entorno definidas en el template por los valores que definimos en las variables exportadas crearemos el archivo de manifiesto de applicacion que desplegará nuestro ambiente aplicativo de negocio - voting-app -.

```
cat previews.yaml | sed -e "s@_APP_ID_@$APP_ID@g" | sed -e "s@_REPO_@$REPO@g" | sed -e "s@_IMAGE_TAG_@$IMAGE_TAG@g" | sed -e "s@_HOSTNAME_@$HOSTNAME@g" | tee helm/templates/$APP_ID.yaml
```

Creamos commit con el nuevo cambio y dejamos que ArgoCD hago su trabajo.

```
cat helm/templates/pr-voting-app-1.yaml

git add .
git commit -m "add pr-1 environment"
git push
```

Podemos sincronizar a mano los objetos o esperar que ArgoCD (si lo hemos definido) sincronice los recursos. ArgoCD por defecto hace un pull de las diferencias cada tres minutos (3 min). 

```
argocd app list
argocd sync list previews
kubectl get ns
kubectl get pods -n pr-voting-app-1
open http://argocd.$INGRESS_HOST.nip.io
```

[SyncOptions](https://argoproj.github.io/argo-cd/user-guide/sync-options/)


### PR-2 

Cremoas un segundo ambiente pero para el branch pr-2. Creamos los artefactos.

```
cd voting-app/
git checkout develop
git checkout -b pr-2 develop
vim vote/templates/index.html
cat helm/values.yaml | sed -e 's@tag: develop@tag: pr-2@g' | helm/values.yaml
cat docker-compose.yml | sed -e 's@:develop@:pr-2@g' | tee docker-compose.yml 
docker-compose up -d 
docker push gonzaloacosta/voting-app-worker:pr-2
docker push gonzaloacosta/voting-app-result:pr-2
docker push gonzaloacosta/voting-app-vote:pr-2
```

Exportamos las variables y creamos la aplicación.

```
export PR_ID=2
export REPO=voting-app
export APP_ID=pr-$REPO-$PR_ID
export IMAGE_TAG=pr-2
export HOSTNAME=$APP_ID.$INGRESS_HOST.nip.io
```

Reemplazamos variables en el template.

```
cat previews.yaml | sed -e "s@_APP_ID_@$APP_ID@g" | sed -e "s@_REPO_@$REPO@g" | sed -e "s@_IMAGE_TAG_@$IMAGE_TAG@g" | sed -e "s@_HOSTNAME_@$HOSTNAME@g" | tee helm/templates/$APP_ID.yaml
```

Realizamos un segundo commit con los cambios.

```
cat helm/templates/pr-voting-app-2.yaml

git add .
git commit -m "add pr-2 environment"
git push
```

Chequeamos el ambiente creado.

```
argocd app list
argocd app sync previews
kubectl get ns
kubectl get pods -n pr-voting-app-2
```

En caso de querer borrar un ambiente solo tendremos que sacar el archivo del repositorio y sincronizar las diferencias.

```
cd gitops-voting-app-previews
rm helm/templates/pr-voting-app-2.yaml

git add .
git commit -m "delete pr-2 environment"
git push

argocd app sync previews
```

Por ultimo, como ArgoCD no elimina el namespace que ha creado para el branch de pr lo eliminamos a mano o desde el pipeline. 

```
kubectl delete namespace pr-voting-app-2
```
## Conclusiones 

Como podemos ver abordar una estrategia de GitOps es una muy buena opción no solo para centralizar el despligue de aplicaciones desde un solo lugar, nuestro repo git, sino tambien para poder mantener un tracking de todos los cambios que en nuestra infra y aplicación podemos hacer. Esto nos lleva a poder tener opciones para poder auditar, trackear y centralizar los cambios y operatoria. Desde el punto de vista de la seguridad es importante y necesario no exponer la api de acceso a nuestros cluster de Kubernetes y dejar que ellos (por ArgoCD) por si solo se encargue de descubrir los cambios y aplicarlos.

## Revisores

- Gonzalo Acosta <gonzalo.acosta@semperti.com>
- Emilio Buchailliot <emilio.buchailliot@semperti.com>
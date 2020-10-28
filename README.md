# Jaeger

Es un sistema de rastreo de aplicaciones, su función es detectar anomalías como latencia o problemas de comunicación en servicios distribuidos mediante trazas.

<br>

## Jaeger Operator (solución Kubernetes)

Para la implementación en Kubernetes existe [Jaeger Operator](https://www.jaegertracing.io/docs/1.20/operator/) el cual es un Operador de Kubernetes que administra la aplicación de forma independiente y personalizada.

<br>

# Instalación del operador

Instalación de CRDs y componentes de jaeger-operator en el namespace observability

```sh
kubectl create namespace observability
kubectl create -n observability -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/crds/jaegertracing.io_jaegers_crd.yaml
kubectl create -n observability -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/service_account.yaml
kubectl create -n observability -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/role.yaml
kubectl create -n observability -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/role_binding.yaml
kubectl create -n observability -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/operator.yaml
```
>**Nota:** Se recomienda descargar antes el archivo `operator.yaml` y configurar la variable `WATCH_NAMESPACE` como vacía, de esto modo se podrán revisar las aplicaciones de todos los Namespaces del cluster.

<br>

Permisos adicionales para el cluster

```sh
kubectl create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/cluster_role.yaml
kubectl create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/cluster_role_binding.yaml
```

<br>

Revisamos el pod de jager-operator

```sh
kubectl -n observability get pod
```

<br>

## Creación de Instancia Jaeger

Las instancias Jaeger que se pueden crear son diversas y dependen de su "estrategia", esto debido a que, se deben instalar los siguientes componentes.

- Jager agent
- Jaeger collector
- Jaeger query
- Jaeger ingestor
- Jaeger UI
 

Las estrategias disponibles son las siguientes.

- allInOne (default)
- production
- streaming

Si no se especifica la estrategía, deployará por defecto "allInOne" ([más información sobre estrategias](https://www.jaegertracing.io/docs/1.20/operator/#deployment-strategies)).
[Aquí podemos encontrar algunos ejemplos de creación de instancias y despliegue del operador](https://github.com/jaegertracing/jaeger-operator/tree/master/examples)

<br>

Una vez que el pod esté corriendo podremos crear nuestra Instancia Jaeger.
La opción más sencilla es instalar todos estos componentes en una sólo imagen (allInOne) como parte de un Deployment de la siguiente manera (recomendado para ambientes bajos o de desarrollo).

```sh
kubectl apply -n observability -f - <<EOF
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: simplest
EOF
```

<br>

Verificamos que el pod esté iniciado

```sh
$ kubectl -n observability get pod |grep simplest
NAME                               READY   STATUS    RESTARTS   AGE
simplest-6b479749bb-brtcw          1/1     Running   0          13s
```

<br>

Como la instalación es allInOne, los componentes de Jaeger se separan en Services

```sh
$ kubectl -n observability get svc |grep simplest
NAME                          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                                  AGE
simplest-agent                ClusterIP   None           <none>        5775/UDP,5778/TCP,6831/UDP,6832/UDP      19s
simplest-collector            ClusterIP   10.43.26.33    <none>        9411/TCP,14250/TCP,14267/TCP,14268/TCP   19s
simplest-collector-headless   ClusterIP   None           <none>        9411/TCP,14250/TCP,14267/TCP,14268/TCP   19s
simplest-query                ClusterIP   10.43.15.126   <none>        16686/TCP                                19s
```

<br>

## Acceso a Jaeger UI

Realizamos un port-forward del service simplest-query.

```sh
$ kubectl -n observability port-forward svc/simplest-query 16686
```
Ahora podremos acceder mediante web http://127.0.0.1:16686

<br>

# Pruebas

Existen dos opciones para poder recolectar traces de una aplicación.

- Agregando un sidecar a cada aplicación (automático o manual).
- Desarrollar la aplicación con la opción de enviar la información hacia una instancia de Jaeger.

<br>

## Ejemplo 1 - Sidecar

Deployaremos una aplicación agregando el siguiente `annotation`. De este modo se agregará automáticamente un Sidecar a la aplicación. Es importante señalar que esto es sólo para recursos `Deployments`, para otros recursos la inyección debe ser manual.

```
sidecar.jaegertracing.io/inject": "true"
```
>**Nota:** el valor "true" es un String por lo que debe ir con las commillas dobles.

<br>

## Despliegue de aplicación (sidecar automático)


```sh
$ kubectl apply -f -<<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    "sidecar.jaegertracing.io/inject": "true"
spec:
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
      annotations:
        sidecar.istio.io/inject: "true"
    spec:
      containers:
      - name: myapp
        image: jaegertracing/vertx-create-span:operator-e2e-tests
        ports:
        - containerPort: 8080
          protocol: TCP
EOF
```

<br>

Verificamos que la aplicación esté activa

```sh
$ kubectl get pod
NAME                      READY   STATUS              RESTARTS   AGE
myapp-85b4f9d75-2p22s     1/1     Running             0          20s
```

Le pasaremos algo de tráfico

```sh
$ kubectl exec -i myapp-85b4f9d75-2p22s  -- sh -c "for i in {1..3}; do curl http://localhost:8080; done"
Defaulting container name to myapp.
Use 'kubectl describe pod/myapp-85b4f9d75-2p22s -n default' to see all of the containers in this pod.
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    18  100    18    0     0    196      0 --:--:-- --:--:-- --:--:--   197Hello from Vert.x!
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    18  100    18    0     0    173      0 --:--:-- --:--:-- --:--:--   174Hello from Vert.x!
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    18  100    18    0     0    221      0 --:--:-- --:--:-- --:--:--   222Hello from Vert.x!
```

Revisamos a través de Jaeger UI y veremos que en la opción "Services" existen los servicios `inventory` y `order` correspondientes a la aplicación de prueba. 
Si seleccionamos una de ellas y ejecutamos "Find Traces" podremos ver la información

<br>

## Agregar sidecar manualmente

Para agregar un sidecard a un recurso `StatefulSet` o `DaemonSet` se debe agregar de forma manual añadiendo las siguientes líneas.

```sh
      - name: jaeger-agent
        image: jaegertracing/jaeger-agent:1.20.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5775
          name: zk-compact-trft
          protocol: UDP
        - containerPort: 5778
          name: config-rest
          protocol: TCP
        - containerPort: 6831
          name: jg-compact-trft
          protocol: UDP
        - containerPort: 6832
          name: jg-binary-trft
          protocol: UDP
        - containerPort: 14271
          name: admin-http
          protocol: TCP
        args:
          - --reporter.grpc.host-port=dns:///simplest-collector-headless.observability.svc:14250
```

---

## Ejemplo 2 - Reporting Spans

La segunda opción es deployar una aplicación que envíe sus trazas hacia el agente de Jaeger, de esta forma no es necesario agregar un sidecar por cada despliegue, sin embargo, esto requiere que la aplicación solicite la dirección del agente y su puerto.

<br>

## Despliegue de aplicación

```sh
kubectl apply -f -<<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/component: hotrod
    app.kubernetes.io/instance: jaeger
  name: jaeger-hotrod
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: hotrod
      app.kubernetes.io/name: jaeger
  template:
    metadata:
      labels:
        app.kubernetes.io/component: hotrod
        app.kubernetes.io/name: jaeger
    spec:
      containers:
      - env:
        - name: JAEGER_AGENT_HOST
          value: simplest-agent.observability.svc.cluster.local
        - name: JAEGER_AGENT_PORT
          value: "6831"
        image: jaegertracing/example-hotrod:latest
        imagePullPolicy: Always
        livenessProbe:
          httpGet:
            path: /
            port: 8080
        name: jaeger-hotrod
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /
            port: 8080
EOF
```

En este ejemplo la aplicación tiene configurada la opción para conectar a un agente Jaeger y los valores se pasan mediante variables JAEGER_AGENT_HOST y JAEGER_AGENT_PORT.

<br>

Al momento de generar tráfico (realizar port-forward al pod de la aplicación) podremos ver en la UI de Jaeger las trazas automáticamente.

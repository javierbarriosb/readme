# Habilitación de Operador grafana para monitoreo de caches

## Despliegue del Operador

### Pre-requisitos

1. Poseer un client OC 
2. Acceso a openshift con usuario con rol cluster-admin
3. Namespace monitoring-infinispan creado
4. Operador Grafana version alpha v4.4.1 habilitado en namespace monitoring-infinispan (namespace donde se administrara el monitoreo)

### 1. Habilitacion de monitoreo en clusteres de caches

1. Login sobre Openshift
```bash
oc login
```
2. Ingresar en proyecto el cual posea los caches a monitorear
```bash
oc project monitoring-data-grid
```
3. Crear un service monitor llamado service-monitor-infinispan
```bash
$ oc apply -f  service-monitor-infinispan.yaml
```
4. Añadir anotacion al a los cluster Infinispan CR *infinispan.org/monitoring* en *true* como se muestra a continuacion
```bash
apiVersion: infinispan.org/v1
kind: Infinispan
metadata:
  name: infinispan
  annotations:
    **infinispan.org/monitoring: 'true'**
```
5. Ingresar a Openshift web console y como administrador, nos dirigimos a nuestro panel, abrimos la pestaña *observe/metrics* confirmando que podemos buscar la siguiente métrica

```bash
vendor_cache_manager_default_cluster_size
```
### 2. Habilitación de user-defined projects 

1. Editamos el *ConfigMap*  llamado cluster-monitoring-config
```bash
oc -n openshift-monitoring edit configmap cluster-monitoring-config
``` 
> Añadimos la variable *enableUserWorkload* en *true* a la escala de data/config.yaml
```bash
 apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    **enableUserWorkload: true**
```
### 3. Habilitación de monitoreo con Grafana 

1. Ingresar a Openshift web console y como administrador, nos dirigimos a *installed operators/grafana*
> Dentro del operador Grafana creamos un Grafana CR (Custom Resource) en la sección superior llamada "grafana"
```bash
apiVersion: integreatly.org/v1alpha1
kind: Grafana
metadata:
  name: grafana-monitoring-infinispan
spec:
  config:
    auth:
      disable_signout_menu: true
    auth.anonymous:
      enabled: true
    log:
      level: warn
      mode: console
    security:
      admin_password: redhat01
      admin_user: admin
  dashboardLabelSelector:
    - matchExpressions:
        - key: monitoring-key
          operator: In
          values:
            - middleware
  ingress:
    enabled: true
``` 
2. Creaamos un serviceAccount para permitir a Grafana leer metricas de data grid
```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: infinispan-monitoring
``` 
> 1. Aplicamos el service account
```bash
oc apply -f service-account.yaml
``` 
> 2. Otorgamos permisos *cluster-monitoring-view* al *ServiceAccount*
```bash
oc adm policy add-cluster-role-to-user cluster-monitoring-view -z infinispan-monitoring
``` 
3. Creamos una fuente de datos de Grafana 
> 1. Obtenemos un token para el ServiceAccount
```bash
oc serviceaccounts get-token infinispan-monitoring
```
> 2. Definimos un data source de tipo Grafana incluyendo el token obtenido anteriormente y lo ingresamos en el archivo a escala de *spec.datasources.secureJsonData.httpHeaderValue1*
```bash
apiVersion: integreatly.org/v1alpha1
kind: GrafanaDataSource
metadata:
  name: infinispan-grafana-datasource
spec:
  name: datasource.yaml
  datasources:
    - access: proxy
      editable: true
      isDefault: true
      jsonData:
        httpHeaderName1: Authorization
        timeInterval: 5s
        tlsSkipVerify: true
      name: Prometheus
    **secureJsonData:**
        httpHeaderValue1: >-
          Bearer
        **eyJhbGciOiJSUzI1NiIsImtpZCI6Ii1LZU...**
      type: prometheus
      url: 'https://thanos-querier.openshift-monitoring.svc.cluster.local:9091'
```
>3. Aplicamos el source 
```bash
oc apply -f grafana-datasource.yaml
```
### 4. Configuración de paneles 
1. Creamos un ConfigMap llamado *infinispan-operator-config* en el operador donde se encuentra nuestros caches 
```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: infinispan-operator-config
data:
  grafana.dashboard.monitoring.key: middleware
  grafana.dashboard.name: infinispan
  grafana.dashboard.namespace: infinispan
``` 
***monitorin.key: es la clave de supervision***


***dashborad.name: es el nombre del grafana dashboard de nuestro operador***


***dashboard.namespace: es el nombre de nuestro name donde se encuentra nuestro cache***

2. Creamos o actualizamos el *infinispan-operator-config* 
```bash
oc apply -f infinispan-operator-config.yaml
```   
3. Obtenemos la ruta url donde se encuentra nuestro Grafana disponible
```bash
oc obtener rutas grafana-ruta -o jsonpath=https://"{.spec.host}"
```  

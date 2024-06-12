# Configuration

### Concert configuration

For the Incident Generator to connect to Concert and read LOS data, a couple of configuration steps are required.

#### Import PredictionLink into Concert (Optional)

On customer system, prediction links should already be present. If not present, you can import prediction links manually using `LinkObjectsImport` menu in Symphony or Concert by following below steps:

1. Go to LinkObjectsImport:
   * In Sympony under: \`Administration -> MDServer -> LinkObjetcsImport\`
   * In Concert under: \`Debug -> MDServer -> LinkObjetcsImport\`
2. Select Object Type: \`PredictionLink\`
3. Import file: It is important to make the prediction-link csv file available for the LinkObjectsImport tool. Unfortunately, this file needs to be avaialble within the clusterfile system first. This can be done via the \`file chooser\` button right to \`Import file\`.It is recommended to create a separate folder (e.g. LinkObjectImport) and upload the required csv files to this folder first.
   * Click \`file chooser -> Upload -> Choose file\`
   * Select the uploaded csv file and click on OK
4. Select Coordinate system: Aus Datei lesen if it is present in csv file
5. Click on Import button
6. Check prediction links: After successful import you should see prediction links
   * In Sympony under: \`Traffic Detection -> Prediction Links\`
   * In Concert under: \`Traffic Data -> Roads -> Prediction Links\`

**Symphony Example**

**Concert Example**

#### OCPI-2 service account

A dedicated service account with appropriate permissions is needed to access Concert via External Interface (OCPI-2). Such an account can be create in Concert under `Settings > User Management > Service Accounts`. Make sure the account is of type `OCPI-2` and has at least the following permissions set:

* TrafficData\_link\_currentLink\_Read
* TrafficMessage\_incidents\_Create
* TrafficMessage\_incidents\_Delete
* TrafficMessage\_incidents\_Write
* TrafficMessage\_incidents\_Read

#### MD Export

The MD Export configuration needs to be extended such that a custom export type is available in Concert. The export will contain prediction link ids, their coordinate location and the projection type.

The required configuration is already prepared as a [configuration file](https://gitlab.mocca.yunextraffic.cloud/its/incident-generator/ocpi2-adapter/-/blob/master/src/main/resources/links/MDObjectsImportExport\_PredictionLinkLocations.xml). This file needs to be copied into Concert's gitea persistence store. The persistence store is available through the URL:

```
https://data-storage.[cluster-domain]
```

where the cluster domain is something like `centrals.eu.dev.yunextraffic.cloud` or `t49.test-brno.com`.

The configuration file must be uploaded to

```
data-as/data/Concert/MDObjectsImportExport/MDObjectsImportExport_PredictionLinkLocations.xml
```

Now a new export type `PredictionLinkLocation` should be available in Concert under `Settings > Special > MD Export/Import`.

#### MD Export Makro

Record a new macro for the `PredictionLinkLocation` export.

* In Concert, navigate to `Control > Strategy management> Macros` and click `Record` to start the recording.
* Then select `Export` from the list and clik `Record` on the top right.
* You will see `PredictionLinkLocation` export window from `Settings > Special > MD Export/Import`.
* Perform MD Export by selecting `PredictionLink` from the list.
* Set export file location to `Public/prediction-links.xml`.
* Click `Export` and save macro with a name and description

NB: In the `its-concert-con-as` container, this folder is located here:

```
/opt/SitrafficData/data/Concert/users/public
```

#### Response Plan

Now we create a responseplan which executes every 24 hours:

1. the macro we recorded above
2. the following command:

```
curl --form file=@prediction-links.xml --form file=@predictions-links.xml http://[incident-generator-svc-name]:8080/uploadFile
```

**Create Program**

In Concert, open the Responseplan Administration dialog via `Settings > Strategy management > Response Plan Administration`, choose tab Program Execution and add a new program with the following properties:

```
ID: curl, Name: Curl Command, Execute Command: /usr/bin/curl, Working Directory: /opt/SitrafficData/data/Concert/users/public, Show: true, Check Result: false
    Name: par1, Text: Parameter 1, Show: true, ActuatorType: Parameter
    Name: par2, Text: Parameter 2, Show: true, ActuatorType: Parameter
    Name: par3, Text: Parameter 3, Show: true, ActuatorType: Parameter
```

click export and check checkbox "Check Result".

**Create Response plan**

Then in Concert `Control > Strategy Management > Response Plans`, create a new response plan with the following attributes:

```
Name: Incident Generator - Update PredictionLinks
    Action 1 - id: mdexport, macro
    Action 2 - id: curl, par1: --form, par2: file=@predictions-links.xml, par3: http://[incident-generator-svc-name]:8080/uploadFile
    Trigger - Time
```

### Interfaces

This section lists the interfaces exposed by the incident generator backend service.

**DMN**

```http
POST /api/incident-generator-backend/dmn/deploy
```

```Request
[
  {
    "id": 64,
    "sortSequence": 1,
    "active": true,
    "predictionLink": "\"newpredic22\"",
    "los": ">5",
    "weekdays": "",
    "startDate": "",
    "endDate": ""
  }
]
```

```Response

"data" : string

```

**Prediction Link**

```http
POST /api/incident-generator-backend/uploadFile
```

```Request
MultiPart File
```

**Rule Controller**

Controller to Save or Update Rules

```http
POST /api/incident-generator-backend/rules
```

```Request
[
  {
    "id": 64,
    "sortSequence": 1,
    "active": true,
    "predictionLink": "\"newpredic22\"",
    "los": ">5",
    "weekdays": "",
    "startDate": "",
    "endDate": ""
  }
]
```

```Response
[
  {
    "id": 64,
    "sortSequence": 1,
    "active": true,
    "predictionLink": "\"newpredic22\"",
    "los": ">5",
    "weekdays": "",
    "startDate": "",
    "endDate": ""
  }
]
```

Controller to get all rules

```http
POST /api/incident-generator-backend/get
```

```Response
[
  {
    "id": 64,
    "sortSequence": 1,
    "active": true,
    "predictionLink": "\"newpredic22\"",
    "los": ">5",
    "weekdays": "",
    "startDate": "",
    "endDate": ""
  }
]
```

### Deployment Configuration (8.3.1 Wildpig)

The below consists a list of variables used by Incident Generator services. The default values can be overridden via helm variables.

<table data-full-width="true"><thead><tr><th>Property</th><th>Description</th><th>Value</th></tr></thead><tbody><tr><td>global.baseDomain</td><td>Domain as per environment</td><td>Default: app.dev.yunex.dev</td></tr><tr><td>its-incident-generator.incident-generator-ui.appDetails.team</td><td>The team which is responsible from this component, used for monitoring purposes</td><td>Default: Obsidian</td></tr><tr><td>its-incident-generator.incident-generator-ui.nameOverride</td><td>Internal flag to build kubernetes service name, changes on this field is not suggested.</td><td>Default: frontend</td></tr><tr><td>its-incident-generator.incident-generator-ui.image.registry</td><td>Docker registry for service image</td><td>Default: its-docker.artifactory.mocca.yunextraffic.cloud</td></tr><tr><td>its-incident-generator.incident-generator-ui.image.pullPolicy</td><td>Defines pull policy for the docker image of application during deployment.</td><td>Default: IfNotPresent</td></tr><tr><td>its-incident-generator.incident-generator-ui.image.imagePullSecrets</td><td>List of the secrets to be used when pulling docker image. The secrets are created during the cluster creation.</td><td><p>Default:</p><pre><code> - name: artifactorypullsecret
</code></pre></td></tr><tr><td>its-incident-generator.incident-generator-ui.podPorts</td><td>Defines exposed port of the application</td><td><p>Default:</p><pre><code>- name: http
  containerPort: 8080
  protocol: TCP
</code></pre></td></tr><tr><td>its-incident-generator.incident-generator-ui.storageClassName</td><td>Storage class name for the created PVCs</td><td>Default: vsphere-csi-sc</td></tr><tr><td>its-incident-generator.incident-generator-ui.sidecarContainers</td><td>Flag to enable and define sidecars</td><td><p>Default:</p><pre><code>enabled: true  
name: "incident-generator.frontend.sidecarContainers"
</code></pre></td></tr><tr><td>its-incident-generator.incident-generator-ui.resources</td><td>Defines initial allocated resources and the resource limits for the application container.</td><td><p>Default:</p><pre><code>requests: 
 cpu: 200m 
 memory: 128Mi 
limits: 
 memory: 256Mi
</code></pre></td></tr><tr><td>its-incident-generator.incident-generator-ui.service</td><td>Kubernetes service definition</td><td><p>Default:</p><pre><code>enabled: true 
type: ClusterIP 
ports: 
  - port: 8080 
    targetPort: 3000 
    protocol: TCP 
<strong>    name: http
</strong></code></pre></td></tr><tr><td>its-incident-generator.incident-generator-ui.gateway</td><td>Kubernetes gateway definition</td><td><p>Default:</p><pre><code>enabled: true
allowInsecureProtocols: false
servers:
  - port:
      name: https
      number: 443
      protocol: HTTPS
    customHosts:
    - "{{- .Values.serviceHost -}}.{{- include "yunex-app.baseDomain" . -}}"
</code></pre></td></tr><tr><td>its-incident-generator.incident-generator-ui.virtualservice</td><td>Kubernetes virtualservice definition</td><td><p>Default:</p><pre><code>enabled: true
customHosts:
  - "{{- .Values.serviceHost -}}.{{- include "yunex-app.baseDomain" . -}}"
</code></pre></td></tr><tr><td>its-incident-generator.incident-generator-ui.clusterDomain</td><td>Defines cluster domain for kubernetes</td><td>Default: svc.cluster.local</td></tr><tr><td>its-incident-generator.incident-generator-ui.serviceHost</td><td>Defines the subdomain in the url for the application. The application url is made by prepending subdomain to global.baseDomain. With default values the application url will look like "incident-generator.app.dev.yunex.dev"</td><td>Default: incident-generator</td></tr><tr><td>its-incident-generator.incident-generator-ui.oauth2</td><td>Defines oauth2 proxy sidecar application</td><td><p>Default:</p><pre><code>cookieExpire: 30m
cookieRefresh: 0
image:
  name: bitnami/oauth2-proxy
  repo: docker-hub.artifactory.mocca.yunextraffic.cloud
  pullPolicy: IfNotPresent
  tag: 7.5.1
resources:
  limits:
    memory: 64Mi
  requests:
    cpu: 100m
    memory: 32Mi
</code></pre></td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.nameOverride</td><td>Internal flag to build kubernetes service name, changes on this field is not suggested.</td><td>Default: ocpi2-adapter</td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.appDetails.team</td><td>The team which is responsible from this component, used for monitoring purposes</td><td>Default: Obsidian</td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.image.registry</td><td>Docker registry for service image</td><td>Default: its-docker.artifactory.mocca.yunextraffic.cloud</td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.image.pullPolicy</td><td>Defines pull policy for the docker image of application during deployment.</td><td>Default: IfNotPresent</td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.image.imagePullSecrets</td><td>List of the secrets to be used when pulling docker image. The secrets are created during the cluster creation.</td><td><p>Default:</p><pre><code> - name: artifactorypullsecret
</code></pre></td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.envVariables</td><td>List of the environment variables set on application, contains endpoint urls, credentials for authorization etc. Should not be changed unless it s required.</td><td><p>Default:</p><pre><code>- name: server.servlet.context-path
value: "{{ .Values.contextPath }}"
- name: POSTGRES_NAME
value: "{{ tpl .Values.postgres.name .}}"
- name: POSTGRES_DB
value: "{{ .Values.postgres.database }}"
- name: PGUSER
valueFrom:
  secretKeyRef:
    name: "{{ tpl .Values.postgres.secret .}}"
    key: postgresql-username
- name: PGPASSWORD
valueFrom:
  secretKeyRef:
    name: "{{ tpl .Values.postgres.secret .}}"
    key: postgresql-password
- name: CAMUNDA_ADMIN_USER
valueFrom:
  secretKeyRef:
    name: "{{ tpl .Values.camunda.secret .}}"
    key: camunda-user
- name: CAMUNDA_ADMIN_PWD
valueFrom:
  secretKeyRef:
    name: "{{ tpl .Values.camunda.secret .}}"
    key: camunda-password
- name: KEEP_LAST_X_DEPLOYMENTS
value: "{{ .Values.camunda.keepLastXDeployments }}"
- name: CONCERT_OCPI_USERNAME
valueFrom:
  secretKeyRef:
    name: "{{ .Values.ocpi2.secret }}"
    key: user
- name: CONCERT_OCPI_PASSWORD
valueFrom:
  secretKeyRef:
    name: "{{ .Values.ocpi2.secret }}"
    key: password
- name: JSON_LOGGER
value: "{{ .Values.logger.json.enabled }}"
</code></pre></td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.podPorts</td><td>Defines exposed port of the application</td><td><p>Default:</p><pre><code>- name: http
  containerPort: 8080
  protocol: TCP
</code></pre></td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.monitoringCfg</td><td>Defines readiness and liveness probes for the monitoring tools</td><td><p>Default:</p><pre><code>readinessProbe:
  httpGet:
    path: /api/incident-generator-backend/actuator/health
    port: 8080
livenessProbe:
  httpGet:
    path: /api/incident-generator-backend/actuator/health/liveness
    port: 8080
  initialDelaySeconds: 180
  periodSeconds: 10
</code></pre></td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.storageClassName</td><td>Storage class name for the created PVCs</td><td>Default: vsphere-csi-sc</td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.sidecarContainers</td><td>Flag to enable and define sidecars</td><td><p>Default:</p><pre><code>enabled: true  
name: "its-incident-generator.backend.sidecarContainers"
</code></pre></td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.resources</td><td>Defines initial allocated resources and the resource limits for the application container.</td><td><p>Default:</p><pre><code>requests: 
 cpu: 200m 
 memory: 512Mi 
limits: 
 memory: 1024Mi
</code></pre></td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.service</td><td>Kubernetes service definition</td><td><p>Default:</p><pre><code>enabled: true
type: ClusterIP
ports:
  - port: 8080
    targetPort: 3000
    protocol: TCP
    name: http
</code></pre></td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.virtualservice</td><td>Kubernetes virtualservice definition</td><td><p>Default:</p><pre><code>virtualservice:
  customGateways:
  - "{{- .Release.Name -}}-frontend-gateway"
  customHosts:
  - "{{- .Values.serviceHost -}}.{{- include "yunex-app.baseDomain" . -}}"
</code></pre></td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.secrets</td><td>Defines the k8s secrets created by this service</td><td><p>Default:</p><pre><code>secrets:
  enabled: true
  data:
    - name: "{{- .Release.Name -}}-camunda-secret"
      stringData:
        camunda-user: "{{- required "Camunda user is not set" .Values.camundaUser -}}"
        camunda-password: "{{- required "Camunda password is not set" .Values.camundaPassword -}}"
</code></pre></td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.clusterDomain</td><td>Defines cluster domain for kubernetes</td><td>Default: svc.cluster.local</td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.serviceHost</td><td>Defines the subdomain in the url for the application. The application url is made by prepending subdomain to global.baseDomain. With default values the application url will look like "incident-generator.app.dev.yunex.dev"</td><td>Default: incident-generator</td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.contextPath</td><td>Context path is a name with which a web application is accessed. It is the root of the application, with default values the endpoint url will be <code>incident-generator.app.dev.yunex.dev/api/incident-generator-backend</code></td><td>Default: /api/incident-generator-backend</td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.postgres</td><td>Defines database connection details. The name here should be overridden incase the postgres service' nameOverride is set to something else than default value.</td><td><p>Default:</p><pre><code>database: incident-generator
secret: "{{- .Release.Name -}}-postgres-secret"
name: "{{- .Release.Name -}}-postgres"
</code></pre></td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.camunda</td><td>Defines camunda secret and number of camunda deployments to be kept in history</td><td><p>Default:</p><pre><code>secret: "{{- .Release.Name -}}-camunda-secret"
keepLastXDeployments: 5
</code></pre></td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.ocpi2.secret</td><td>k8s secret name to connect OCPI2 endpoint of Concert</td><td>Default: keycloak-extif-incident-generator</td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.devMode</td><td>Flag to enable development mode, used to disable cors check on development clusters</td><td>Default: false</td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.oauth2</td><td>Defines oauth2 proxy sidecar application</td><td><p>Default:</p><pre><code>cookieExpire: 30m
cookieRefresh: 0
image:
  name: bitnami/oauth2-proxy
  repo: docker-hub.artifactory.mocca.yunextraffic.cloud
  tag: 7.5.1
  pullPolicy: IfNotPresent
resources:
  limits:
    memory: 64Mi
  requests:
    cpu: 100m
    memory: 32Mi
</code></pre></td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.logger.json.enabled</td><td>Defines json format application logging.</td><td>Default: true</td></tr><tr><td>its-incident-generator.its-incident-generator-postgresql.nameOverride</td><td>Internal flag to build kubernetes service name, changes on this field is not suggested. Incase the default value is overridden, the name value in its-incident-generator.its-incident-generator-ocpi2-adapter.postgres field should be configured properly.</td><td>Default: postgres</td></tr><tr><td>its-incident-generator.its-incident-generator-postgresql.appDetails.team</td><td>The team which is responsible from this component, used for monitoring purposes</td><td>Default: Obsidian</td></tr><tr><td>its-incident-generator.its-incident-generator-postgresql.image.pullPolicy</td><td>Defines pull policy for the docker image of application during deployment.</td><td>Default: IfNotPresent</td></tr><tr><td>its-incident-generator.its-incident-generator-postgresql.storageClassName</td><td>Storage class for the application PVCs</td><td>Default: vsphere-csi-sc</td></tr><tr><td>its-incident-generator.its-incident-generator-postgresql.backupStorageClassName</td><td>Storage class for the PVCs which are backed up by Velero</td><td>Default: nfs-backup</td></tr><tr><td>its-incident-generator.its-incident-generator-postgresql.disableDeployment</td><td>Flag to disable deployment via wp-library since it s done via Bitnami scripts</td><td>Default: true</td></tr><tr><td>its-incident-generator.its-incident-generator-postgresql.secrets</td><td>Defines the k8s secrets to be used for DB authentication</td><td><p>Default:</p><pre><code>enabled: true
data:
  - name: "{{- .Release.Name -}}-postgres-secret"
    stringData:
      postgresql-password: "{{- required "PostgreSQL password is not set" .Values.postgresqlPassword -}}"
      postgresql-username: postgres
</code></pre></td></tr><tr><td>its-incident-generator.its-incident-generator-postgresql.persistence</td><td>Defines persistence for the pod and backup option for the PVC</td><td><p>Default:</p><pre><code>data:
  size: 8Gi
  backup: true
</code></pre></td></tr><tr><td>its-incident-generator.its-incident-generator-postgresql.volumeFromConfigMap</td><td>Used to create volume from the config map where the init scripts of the database is located</td><td><p>Default:</p><pre><code>- configMap: init-sql
  data:
    init.sql: |-
      CREATE DATABASE incident-generator;
      GRANT ALL PRIVILEGES ON DATABASE incident-generator to postgres;
</code></pre></td></tr><tr><td>its-incident-generator.its-incident-generator-postgresql.postgresql</td><td>Bitnami chart configuration is defined here. PostgreSQL is installed via Bitnami helm chart, and Bitnami specific deployment configuration is defined under this field.</td><td><p>Default:</p><pre><code>global:
  postgresql:
    auth:
      database: incident-generator
      existingSecret: "{{- .Release.Name -}}-postgres-secret"
      secretKeys:
        adminPasswordKey: "postgresql-password"
nameOverride: postgres
commonAnnotations:
  yunex.app/fullname: "{{- include "yunex-app.fullname" . -}}"
image:
  registry: docker-hub.artifactory.mocca.yunextraffic.cloud
  pullPolicy: IfNotPresent
  pullSecrets:
    - artifactorypullsecret
commonLabels:
  yunex.app/fullname: "{{- include "yunex-app.fullname" . -}}"
primary:
  initdb:
    scriptsConfigMap: "{{- include "yunex-app.fullname" . -}}-init-sql"
  resources:
    requests:
      cpu: 200m
      memory: 128Mi
    limits:
      memory: 256Mi
  persistence:
    enabled: true
    existingClaim: "{{- include "yunex-app.fullname" . -}}-data-pvc"
  podAnnotations:
    backup.velero.io/backup-volumes: data
  extraVolumes:
    - name: postgres-backup
      persistentVolumeClaim:
        claimName: "{{- include "yunex-app.fullname" . -}}-postgres-backup-pvc"
  extraVolumeMounts:
    - name: postgres-backup
      mountPath: /backup
</code></pre></td></tr><tr><td>its-incident-generator.its-incident-generator-postgresql.postgresBackup</td><td>Used to enable backup on database</td><td><p>Default:</p><pre><code>enabled: true
schedule: "0 2 * * *"
access:
  secretName: "{{- .Release.Name -}}-postgres-secret"
  passwordKey: postgresql-password
  usernameKey: postgresql-username
servicePort: 5432
storageSize: "8Gi"
retention: 6
image:
  registry: docker-hub.artifactory.mocca.yunextraffic.cloud
  repository: bitnami/postgresql
  tag: 14.3.0-debian-10-r17
</code></pre></td></tr></tbody></table>

### Deployment Configuration (8.3.0)

The below consists a list of variables used by Incident Generator services. The default values can be overridden via helm variables.

<table data-full-width="true"><thead><tr><th>Property</th><th>Description</th><th>Value</th></tr></thead><tbody><tr><td>global.baseDomain</td><td>Domain as per environment</td><td>Default: centrals.eu.dev.yunextraffic.cloud</td></tr><tr><td>global.oauth2.cookieRefresh</td><td>Cookie refresh seconds, 0 means disabled. If the application is used together with SAF, the cookie refresh is disabled because token refresh is handled in SAF.</td><td>Default: 0</td></tr><tr><td>global.oauth2.cookieExpire</td><td>Cookie expire time</td><td>Default: 168h</td></tr><tr><td>global.yunex.repositories.itsDocker</td><td>Docker repository for ITS in house components</td><td>Default: its-docker.artifactory.mocca.yunextraffic.cloud</td></tr><tr><td>global.yunex.repositories.dockerHub</td><td>Proxy url for official docker repository, proxy is used since some of the environments might not have direct internet connection. Used to download docker images from official docker-hub.</td><td>Default: docker-hub.artifactory.mocca.yunextraffic.cloud</td></tr><tr><td>global.yunex.repositories.quay</td><td>Proxy url for Quay docker repository, proxy is used since some of the environments might not have direct internet connection. Used to download docker images from Quay repository.</td><td>Default: quay.artifactory.mocca.yunextraffic.cloud</td></tr><tr><td>global.yunex.imagePullSecrets</td><td>List of secrets for the docker repositories mentioned. Each repository has it is own kubernetes secret which contain credentials. These secrets are created automatically during the cluster creation.</td><td><p>Default:</p><pre><code>- name: itsdockerpullsecret
- name: dockerhubpullsecret
- name: quaypullsecret
</code></pre></td></tr><tr><td>its-incident-generator.incident-generator-ui.image.pullPolicy</td><td>Defines pull policy for the docker image of application during deployment.</td><td>Default: Always</td></tr><tr><td>its-incident-generator.incident-generator-ui.resources</td><td>Defines initial allocated resources and the resource limits for the application container.</td><td><p>Default:</p><pre><code>requests: 
 cpu: 200m 
 memory: 128Mi 
limits: 
 memory: 256Mi
</code></pre></td></tr><tr><td>its-incident-generator.incident-generator-ui.gatekeeperImage</td><td>Defines oauth2 proxy sidecar application</td><td><p>Default:</p><pre><code>name: bitnami/oauth2-proxy
tag: 7.5.1
pullPolicy: IfNotPresent
resources:
  limits:
    memory: 64Mi
  requests:
    cpu: 100m
    memory: 32Mi
</code></pre></td></tr><tr><td>its-incident-generator.incident-generator-ui.subdomain</td><td>Defines the subdomain in the url for the application. The application url is made by prepending subdomain to global.baseDomain. With default values the application url will look like "incident-generator.centrals.eu.dev.yunextraffic.cloud"</td><td>Default: incident-generator</td></tr><tr><td>its-incident-generator.incident-generator-ui.clusterDomain</td><td>Defines cluster domain for kubernetes</td><td>Default: svc.cluster.local</td></tr><tr><td>its-incident-generator.incident-generator-ui.istio</td><td>Flag to enable istio related resource creation, should be disabled on the environment which does not have istio</td><td>Default: true</td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.postgres</td><td>Defines database connection details</td><td><p>Default:</p><pre><code>database: incident-generator
secret: its-incident-generator-postgres-secret
name: its-incident-generator-postgres
</code></pre></td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.camunda</td><td>Defines camunda secret and number of camunda deployments to be kept in history</td><td><p>Default:</p><pre><code>secret: its-incident-generator-camunda-secret
keepLastXDeployments: 5
</code></pre></td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.resources</td><td>Defines initial allocated resources and the resource limits for the application container.</td><td><p>Default:</p><pre><code>requests: 
 cpu: 200m 
 memory: 512Mi 
limits: 
 memory: 1024Mi
</code></pre></td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.subdomain</td><td>Defines the subdomain in the url for the application. The application url is made by prepending subdomain to global.baseDomain. With default values the application url will look like "incident-generator.centrals.eu.dev.yunextraffic.cloud"</td><td>Default: incident-generator</td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.gateway</td><td>Kubernetes gateway to be used by application</td><td>Default: incident-generator-ui-gateway</td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.contextPath</td><td>Context path is a name with which a web application is accessed. It is the root of the application, with default values the endpoint url will be <code>incident-generator.centrals.eu.dev.yunextraffic.cloud/api/incident-generator-backend</code></td><td>Default: /api/incident-generator-backend</td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.clusterDomain</td><td>Defines cluster domain for kubernetes</td><td>Default: svc.cluster.local</td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.gatekeeperImage</td><td>Defines oauth2 proxy sidecar application</td><td><p>Default:</p><pre><code>name: bitnami/oauth2-proxy
tag: 7.5.1
pullPolicy: IfNotPresent
resources:
  limits:
    memory: 64Mi
  requests:
    cpu: 100m
    memory: 32Mi
</code></pre></td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.devMode</td><td>Flag to enable development mode, used to disable cors check on development clusters</td><td>Default: false</td></tr><tr><td>its-incident-generator.its-incident-generator-ocpi2-adapter.logger.json.enabled</td><td>Defines json format application logging.</td><td>Default: true</td></tr><tr><td>its-incident-generator.its-incident-generator-postgresql.clusterDomain</td><td>Defines cluster domain for kubernetes</td><td>Default: svc.cluster.local</td></tr><tr><td>its-incident-generator.its-incident-generator-postgresql.postgresql</td><td>Bitnami chart configuration is defined here. PostgreSQL is installed via Bitnami helm chart, and Bitnami specific deployment configuration is defined under this field.</td><td><p>Default:</p><pre><code>global:
  postgresql:
    auth:
      database: incident-generator
      existingSecret: its-incident-generator-postgres-secret
      secretKeys:
        adminPasswordKey: "postgresql-password"
fullnameOverride: its-incident-generator-postgres
nameOverride: its-incident-generator-postgres
primary:
  initdb:
    scriptsConfigMap: its-incident-generator-postgres-init-sql
  resources:
    requests:
      cpu: 200m
      memory: 128Mi
    limits:
      memory: 256Mi
</code></pre></td></tr></tbody></table>

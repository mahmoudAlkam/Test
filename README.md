Hier ist eine detaillierte Erklärung der Hauptfunktionen und Aufgaben jedes der Skripte in der Kubernetes-Konfiguration für Fluent Bit, Loki, Grafana und Traefik:

### Fluent Bit

#### ConfigMap: `fluentbit-parsers`
- **Hauptfunktion**: Definiert Parser für Fluent Bit, um das Log-Format zu analysieren.
- **Aufgaben**:
  - Legt einen JSON-Parser fest, der die Zeit im Log-Eintrag identifiziert und im richtigen Format speichert.
- **Detaillierte Erklärung**:
  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: fluentbit-parsers
    namespace: local
  data:
    parsers.conf: |
      [PARSER]
          Name   json
          Format json
          Time_Key time
          Time_Format %Y-%m-%dT%H:%M:%S.%L
          Time_Keep On
  ```
  - **[PARSER]**: Definiert einen Parser namens `json`, der das Format JSON erwartet.
  - **Time_Key**: Gibt das Zeit-Schlüsselwort an, um Zeitstempel im JSON zu erkennen.
  - **Time_Format**: Gibt das Zeitformat an, das verwendet wird.
  - **Time_Keep**: Behält die ursprüngliche Zeit bei.

#### ConfigMap: `fluentbit-config`
- **Hauptfunktion**: Konfiguriert Fluent Bit, um Logs zu sammeln, zu filtern und an Loki weiterzuleiten.
- **Aufgaben**:
  - Definiert die Service-Einstellungen, Input, Filter und Output für Fluent Bit.
- **Detaillierte Erklärung**:
  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: fluentbit-config
    namespace: local
  data:
    fluent-bit.conf: |
      [SERVICE]
          Flush        5
          Daemon       Off
          Log_Level    info
          Parsers_File parsers.conf

      [INPUT]
          Name              tail
          Tag               solair.*
          Path              /var/log/solair/service/*.log
          Parser            json
          DB                /var/log/flb_kube.db
          Mem_Buf_Limit     5MB
          Skip_Long_Lines   On

      [FILTER]
          Name                kubernetes
          Match               solair.*
          Merge_Log           On
          Keep_Log            Off
          K8S-Logging.Parser  On
          K8S-Logging.Exclude On

      [OUTPUT]
          Name            loki
          Match           *
          Host            loki.local.svc.cluster.local
          Port            3100
          Labels          {job="fluentbit", instance="solair"}
          Line_Format     json
  ```
  - **[SERVICE]**: Globale Einstellungen wie Flush-Intervall, Daemon-Modus und Log-Level.
  - **[INPUT]**: Definiert die Eingabequelle als Log-Dateien im Verzeichnis `/var/log/solair/service/*.log`.
  - **[FILTER]**: Verarbeitet Logs, die von Kubernetes kommen, um zusätzliche Metadaten hinzuzufügen und Logs zusammenzuführen.
  - **[OUTPUT]**: Leitet die verarbeiteten Logs an Loki weiter.

#### DaemonSet: `fluentbit`
- **Hauptfunktion**: Stellt sicher, dass Fluent Bit auf jedem Knoten im Cluster läuft.
- **Aufgaben**:
  - Konfiguriert einen DaemonSet, um Fluent Bit auf jedem Knoten zu deployen.
- **Detaillierte Erklärung**:
  ```yaml
  apiVersion: apps/v1
  kind: DaemonSet
  metadata:
    name: fluentbit
    namespace: logging
    labels:
      k8s-app: fluentbit
  spec:
    selector:
      matchLabels:
        k8s-app: fluentbit
    template:
      metadata:
        labels:
          k8s-app: fluentbit
      spec:
        containers:
        - name: fluentbit
          image: fluent/fluent-bit:latest
          volumeMounts:
          - name: varlog
            mountPath: /var/log
          - name: config
            mountPath: /fluent-bit/etc/fluent-bit.conf
            subPath: fluent-bit.conf
          - name: parsers
            mountPath: /fluent-bit/etc/parsers.conf
            subPath: parsers.conf
          securityContext:
            runAsUser: 0
        volumes:
          - name: varlog
            hostPath:
              path: /var/log
              type: DirectoryOrCreate
          - name: config
            configMap:
              name: fluentbit-config
          - name: parsers
            configMap:
              name: fluentbit-parsers
  ```
  - **volumeMounts**: Bindet Verzeichnisse und Dateien an Container.
  - **securityContext**: Setzt Sicherheitskontext für den Container.

### Loki

#### ConfigMap: `loki-config`
- **Hauptfunktion**: Konfiguriert Loki für den Empfang, die Speicherung und das Abfragen von Logs.
- **Aufgaben**:
  - Definiert die Einstellungen für den Server, Ingestor, Schema, Speicher und Limits.
- **Detaillierte Erklärung**:
  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: loki-config
    namespace: logging
  data:
    loki-local-config.yaml: |
      auth_enabled: false

      server:
        http_listen_port: 3100

      ingester:
        lifecycler:
          address: 127.0.0.1
          ring:
            kvstore:
              store: inmemory
            replication_factor: 1
          final_sleep: 0s
        chunk_idle_period: 3m
        chunk_retain_period: 1m
        max_transfer_retries: 0
        wal:
          enabled: true
          dir: /wal  
          
      schema_config:
        configs:
          - from: 2020-10-24
            store: boltdb
            object_store: filesystem
            schema: v11
            index:
              prefix: index_
              period: 168h

      storage_config:
        boltdb:
          directory: /loki/index

        filesystem:
          directory: /loki/chunks

      limits_config:
        enforce_metric_name: false
        reject_old_samples: true
        reject_old_samples_max_age: 168h

      chunk_store_config:
        max_look_back_period: 0s

      table_manager:
        retention_deletes_enabled: false
        retention_period: 0s
  ```
  - **server**: Konfiguriert den HTTP-Server.
  - **ingester**: Einstellungen für das Schreiben und Speichern von Logs.
  - **schema_config**: Definiert das Schema und die Index-Einstellungen.
  - **storage_config**: Legt die Speicherorte für Indexe und Chunks fest.
  - **limits_config**: Setzt Beschränkungen für die Annahme von Logs.
  - **chunk_store_config**: Konfiguration für den Chunk-Speicher.
  - **table_manager**: Verwaltung der Tabellen.

#### StatefulSet: `loki`
- **Hauptfunktion**: Stellt sicher, dass Loki als zustandsbehafteter Dienst betrieben wird.
- **Aufgaben**:
  - Konfiguriert StatefulSet, um sicherzustellen, dass Loki mit persistentem Speicher arbeitet.
- **Detaillierte Erklärung**:
  ```yaml
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: loki
    namespace: logging
  spec:
    serviceName: loki
    replicas: 1
    selector:
      matchLabels:
        app: loki
    template:
      metadata:
        labels:
          app: loki
      spec:
        securityContext:
          runAsUser: 1000
          runAsGroup: 3000
          fsGroup: 2000  # Setzt die Gruppen-ID, damit Loki Zugriff auf die Volumes hat
        containers:
        - name: loki
          image: grafana/loki:2.9.8
          args:
            - -config.file=/etc/loki/loki-local-config.yaml
          ports:
            - containerPort: 3100
          volumeMounts:
            - name: config
              mountPath: /etc/loki
            - name: storage
              mountPath: /loki
            - name: wal
              mountPath: /wal
        volumes:
          - name: config
            configMap:
              name: loki-config
          - name: storage
            emptyDir: {}
          - name: wal
            persistentVolumeClaim:
              claimName: loki-pvc
  ```
  - **volumeMounts**: Bindet die Konfiguration und Speicherorte an den Container.
  - **securityContext**: Setzt Sicherheitskontexte für Benutzer- und Gruppenrechte.

#### PersistentVolumeClaim: `loki-pvc`
- **Hauptfunktion**: Bereitstellung von persistentem Speicher für Loki.
- **Aufgaben**:
  - Definiert die Anforderungen an den Speicher für Loki.
- **Detaillierte Erklärung**:
  ```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: loki-pvc
    namespace: logging
  spec:
    accessModes:
      - ReadWriteOnce
    resources

:
      requests:
        storage: 10Gi
  ```
  - **accessModes**: Gibt an, dass der Speicher nur einmal beschreibbar ist.
  - **resources**: Fordert 10 GB Speicherplatz an.

### Grafana

#### Deployment: `grafana`
- **Hauptfunktion**: Deployment von Grafana für die Visualisierung von Logs.
- **Aufgaben**:
  - Stellt sicher, dass Grafana mit der richtigen Konfiguration und Verbindung zu Loki läuft.
- **Detaillierte Erklärung**:
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: grafana
    namespace: monitoring
    labels:
      app: grafana
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: grafana
    template:
      metadata:
        labels:
          app: grafana
      spec:
        containers:
        - name: grafana
          image: grafana/grafana:9.2.1
          ports:
            - containerPort: 3000
          volumeMounts:
            - name: grafana-storage
              mountPath: /var/lib/grafana
            - name: grafana-datasource
              mountPath: /etc/grafana/provisioning/datasources
        volumes:
          - name: grafana-storage
            emptyDir: {}
          - name: grafana-datasource
            configMap:
              name: grafana-datasource
  ```
  - **volumeMounts**: Bindet Speicher und Datenquellen-Konfiguration an den Container.

#### ConfigMap: `grafana-datasource`
- **Hauptfunktion**: Konfiguriert die Datenquelle in Grafana, um Logs von Loki zu beziehen.
- **Aufgaben**:
  - Definiert die Datenquelle für Loki in Grafana.
- **Detaillierte Erklärung**:
  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: grafana-datasource
    namespace: monitoring
  data:
    datasources.yaml: |
      apiVersion: 1
      datasources:
      - name: Loki
        type: loki
        access: proxy
        url: http://loki.logging.svc.cluster.local:3100
        isDefault: true
  ```
  - **datasources**: Definiert die URL und andere Einstellungen für die Loki-Datenquelle.

#### Service: `grafana`
- **Hauptfunktion**: Stellt einen Dienst für den Zugriff auf Grafana bereit.
- **Aufgaben**:
  - Definiert den Zugriffspunkt für Grafana im Cluster.
- **Detaillierte Erklärung**:
  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: grafana
    namespace: monitoring
  spec:
    ports:
      - port: 3000
        targetPort: 3000
    selector:
      app: grafana
  ```
  - **ports**: Legt den Port fest, über den auf Grafana zugegriffen werden kann.

### Traefik

#### Ingress: `component-service`
- **Hauptfunktion**: Routing der Anfragen zu den entsprechenden Diensten im Cluster.
- **Aufgaben**:
  - Definiert, wie Anfragen an verschiedene Pfade im Cluster geroutet werden.
- **Detaillierte Erklärung**:
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: component-service
    annotations:
      kubernetes.io/ingress.class: "traefik"
      traefik.ingress.kubernetes.io/router.entrypoints: websecure
      traefik.ingress.kubernetes.io/router.tls: "true"
  spec:
    tls: 
    - hosts:
      - oss-les.corp.capgemini.com
      secretName: ingress-tls-secret
    rules:
    - host: oss-les.corp.capgemini.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: component-service
              port:
                number: 8080
        - path: /source
          pathType: Prefix
          backend:
            service:
              name: nginx-service
              port:
                number: 80
  ```
  - **rules**: Definiert, wie Anfragen an verschiedene Pfade weitergeleitet werden.

#### Middleware: `redirect`
- **Hauptfunktion**: Automatische Umleitung von HTTP zu HTTPS.
- **Aufgaben**:
  - Stellt sicher, dass alle Anfragen über HTTPS erfolgen.
- **Detaillierte Erklärung**:
  ```yaml
  apiVersion: traefik.containo.us/v1alpha1
  kind: Middleware
  metadata:
    name: redirect
  spec:
    redirectScheme:
      scheme: https
      permanent: true
  ```
  - **redirectScheme**: Definiert die Umleitungsschemata.

#### Ingress und Middleware für ElasticMQ
- **Hauptfunktion**: Spezifisches Routing und Middleware-Konfiguration für ElasticMQ.
- **Aufgaben**:
  - Definiert das Routing, die Authentifizierung und andere Middleware für ElasticMQ.
- **Detaillierte Erklärung**:
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: elasticmq-ingress
    annotations:
      kubernetes.io/ingress.class: "traefik"
      traefik.ingress.kubernetes.io/router.middlewares: prod-elasticmq-chain@kubernetescrd
      traefik.ingress.kubernetes.io/router.entrypoints: websecure
      traefik.ingress.kubernetes.io/router.tls: "true"
  spec:
    tls: 
    - hosts:
      - oss-les.corp.capgemini.com
      secretName: ingress-tls-secret
    rules:
    - host: oss-les.corp.capgemini.com
      http:
        paths:
        - path: /static
          pathType: Prefix
          backend:
            service:
              name: elasticmq-service
              port:
                number: 9325
        - path: /elasticmq-ui
          pathType: Prefix
          backend:
            service:
              name: elasticmq-service
              port:
                number: 9325
  ```

#### Chain Middleware: `elasticmq-chain`
- **Hauptfunktion**: Ketten von mehreren Middleware-Instanzen für ElasticMQ.
- **Aufgaben**:
  - Kombination von Authentifizierung, Prefix-Entfernung und URL-Anpassung.
- **Detaillierte Erklärung**:
  ```yaml
  apiVersion: traefik.containo.us/v1alpha1
  kind: Middleware
  metadata:
    name: elasticmq-chain
  spec:
    chain:
      middlewares:
      - name: elasticmq-auth
      - name: add-trailing-slash-elasticmq-ui
      - name: elasticmq-stripprefix
  ```

#### Auth Middleware: `elasticmq-auth`
- **Hauptfunktion**: Basis-Authentifizierung für ElasticMQ.
- **Aufgaben**:
  - Schutz des Zugriffs auf ElasticMQ durch Basis-Authentifizierung.
- **Detaillierte Erklärung**:
  ```yaml
  apiVersion: traefik.containo.us/v1alpha1
  kind: Middleware
  metadata:
    name: elasticmq-auth
  spec:
    basicAuth:
      secret: elasticmq-auth
  ```

#### Strip Prefix Middleware: `elasticmq-stripprefix`
- **Hauptfunktion**: Entfernt den URL-Prefix `/elasticmq-ui`.
- **Aufgaben**:
  - Anpassung der URL für den Zugriff auf ElasticMQ-UI.
- **Detaillierte Erklärung**:
  ```yaml
  apiVersion: traefik.containo.us/v1alpha1
  kind: Middleware
  metadata:
    name: elasticmq-stripprefix
  spec:
    stripPrefix:
      prefixes:
        - /elasticmq-ui
  ```

#### Add Trailing Slash Middleware: `add-trailing-slash-elasticmq-ui`
- **Hauptfunktion**: Fügt ein abschließendes `/` zur URL hinzu.
- **Aufgaben**:
  - Sicherstellen, dass URLs korrekt formatiert sind.
- **Detaillierte Erklärung**:
  ```yaml
  apiVersion: traefik.containo.us/v1alpha1
  kind: Middleware
  metadata:
    name: add-trailing-slash-elasticmq-ui
  spec:
    redirectRegex:
      permanent: true
      regex: "^(.+)://(.+)/elasticmq-ui$"
      replacement: "${1}://${2}/elasticmq-ui/"
  ```

Diese Konfiguration stellt sicher, dass Logs gesammelt, verarbeitet, gespeichert und visualisiert werden können, während die gesamte Kommunikation und der Zugang sicher und effizient verwaltet werden.

# k6-telegraf-influxdb-grafana-k6
k6s-telegraf-influxdb-grafana on Kubernetes is a streamlined solution for monitoring and performance testing in Kubernetes. It combines k6, Telegraf, InfluxDB, and Grafana to provide scalable and customizable monitoring capabilities.

## InfluxDB

### InfluxDB Secret

	kubectl create secret generic influxdb-creds --from-literal=INFLUXDB_DB=monitoring --from-literal=INFLUXDB_USER=user --from-literal=INFLUXDB_USER_PASSWORD=password --from-literal=INFLUXDB_READ_USER=readonly --from-literal=INFLUXDB_ADMIN_USER=root --from-literal=INFLUXDB_ADMIN_USER_PASSWORD=password --from-literal=INFLUXDB_HOST=influxdb  --from-literal=INFLUXDB_HTTP_AUTH_ENABLED=true

### InfluxDB PVC

	---
	apiVersion: v1
	kind: PersistentVolumeClaim
	metadata:
	  namespace: monitoring
	  labels:
	    app: influxdb
	  name: influxdb-pvc
	spec:
	  accessModes:
	  - ReadWriteOnce
	  resources:
	    requests:
	      storage: 5Gi

### InfluxDB SVC

	apiVersion: v1
	kind: Service
	metadata:
	  labels:
	    app: influxdb
	  name: influxdb
	  namespace: monitoring
	spec:
	  ports:
	  - port: 8086
	    protocol: TCP
	    targetPort: 8086
	  selector:
	    app: influxdb
	  type: LoadBalancer

### InfluxDB Deployemnt:

	---
	apiVersion: apps/v1
	kind: Deployment
	metadata:
	  namespace: monitoring
	  labels:
	    app: influxdb
	  name: influxdb
	spec:
	  replicas: 1
	  selector:
	    matchLabels:
	      app: influxdb
	  template:
	    metadata:
	      labels:
	        app: influxdb
	    spec:
	      containers:
	      - envFrom:
	        - secretRef:
	            name: influxdb-creds
	        image: docker.io/influxdb:1.8
	        name: influxdb
	        volumeMounts:
	        - mountPath: /var/lib/influxdb
	          name: var-lib-influxdb
	        #resources:
	        #  limits:
	        #    cpu: 300m
	        #    memory: 2Gi
	        #  requests:
	        #    cpu: 105m
	        #    memory: 1Gi
	      volumes:
	      - name: var-lib-influxdb
	        persistentVolumeClaim:
	          claimName: influxdb-pvc

### Deploy the yaml files:

	kubectl apply -f influxdb-deployment.yaml
	kubectl apply -f influx-pvc.yaml
	kubectl apply -f influxdb-svc.yaml

### Expose the service

	minikube service influx

## Telegraf with statsd plugin

### Telegraf ConfigMap

	apiVersion: v1
	kind: ConfigMap
	metadata:
	  name: telegraf-config
	  namespace: monitoring
	data:
	  telegraf.conf: |+
	    [[outputs.influxdb]]
	      urls = ["$http://$INFLUXDB_HOST:8086/"]
	      database = "$INFLUXDB_DB"
	      username = "$INFLUXDB_USER"
	      password = "$INFLUXDB_USER_PASSWORD"
	    [[inputs.statsd]]
	      max_tcp_connections = 250
	      tcp_keep_alive = false
	      service_address = ":8125"
	      delete_gauges = true
	      delete_counters = true
	      delete_sets = true
	      delete_timings = true
	      metric_separator = "."
	      allowed_pending_messages = 10000
	      percentile_limit = 1000
	      parse_data_dog_tags = true 
	      read_buffer_size = 65535	

### Telegraf Service

	kubectl expose deployment telegraf --port=8125 --target-port=8125 --protocol=UDP --type=NodePort

	minikube service telegraf --namespace monitoring

### Telegraf Deployment

	---
	
	apiVersion: apps/v1
	kind: Deployment
	metadata:
	  namespace: monitoring
	  name: telegraf
	spec:
	  selector:
	    matchLabels:
	      app: telegraf
	  minReadySeconds: 5
	  template:
	    metadata:
	      labels:
	        app: telegraf
	    spec:
	      containers:
	        - image: telegraf:1.10.0
	          name: telegraf
	          envFrom:
	            - secretRef:
	                name: influxdb-creds
	          volumeMounts:
	            - name: telegraf-config-volume
	              mountPath: /etc/telegraf/telegraf.conf
	              subPath: telegraf.conf
	              readOnly: true
	          resources:
	            limits:
	              cpu: 100m
	              memory: 128Mi
	            requests:
	              cpu: 100m
	              memory: 128Mi
	      volumes:
	        - name: telegraf-config-volume
	          configMap:
	            name: telegraf-config

### Deploy the yaml files:

	kubectl apply -f telegraf-configmap.yaml
	kubectl apply -f telegraf-deployment.yaml
	influxdb-svc.yaml

## Grafana

### Grafana Secret

	kubectl create secret generic grafana-creds --from-literal=GF_SECURITY_ADMIN_USER=admin --from-literal=GF_SECURITY_ADMIN_PASSWORD=admin

### Grafana Deployment

	apiVersion: apps/v1
	kind: Deployment
	metadata:
	  namespace: monitoring
	  annotations:
	  creationTimestamp: null
	  generation: 1
	  labels:
	    app: grafana
	  name: grafana
	spec:
	  progressDeadlineSeconds: 600
	  replicas: 1
	  revisionHistoryLimit: 10
	  selector:
	    matchLabels:
	      app: grafana
	  strategy:
	    rollingUpdate:
	      maxSurge: 25%
	      maxUnavailable: 25%
	    type: RollingUpdate
	  template:
	    metadata:
	      creationTimestamp: null
	      labels:
	        app: grafana
	    spec:
	      containers:
	      - envFrom:
	        - secretRef:
	            name: grafana-creds
	        image: docker.io/grafana/grafana:9.3.8
	        imagePullPolicy: IfNotPresent
	        name: grafana
	        resources: {
	          "limits": {
	            "cpu": "100m",
	            "memory": "128Mi"
	          },
	          "requests": {
	            "cpu": "100m",
	            "memory": "128Mi"
	          }
	        }
	        terminationMessagePath: /dev/termination-log
	        terminationMessagePolicy: File
	      dnsPolicy: ClusterFirst
	      restartPolicy: Always
	      schedulerName: default-scheduler
	      securityContext: {}
	      terminationGracePeriodSeconds: 30

### For future reference a persistent volume should be added to grafana

### Expose Grafana

	kubectl expose deployment grafana --type=LoadBalancer --port=3000 --target-port=3000 --protocol=TCP

### import graph

+ Sign in
+ Click on the setting icon and click on data sources
+ Choose influxdb
+ Add the necessary config:
	* ipaddress:port
	* username
	* password
+ save
+ Goto dashboards and import the following ID 2587 to get the k6 loadtesting results

## k6

### Installation

	sudo gpg -k
	sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
	echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] <https://dl.k6.io/deb> stable main" | sudo tee /etc/apt/sources.list.d/k6.list
	sudo apt-get update
	sudo apt-get install k6

### simple script

	K6 run script.js --insecure-skip-tls-verify

		import http from 'k6/http';
		import { sleep } from 'k6';
		
		export default function () {
		  http.get('https://test.k6.io');
		  sleep(1);
		}

### Test k6

	k6 run --out influxdb=http://username:password@IP:PORT/DBNAME simple.js --insecure-skip-tls-verify --vus 10 --duration 2m --no-summary

## References

[docker-compose version](https://medium.com/@nairgirish100/k6-with-docker-compose-influxdb-grafana-344ded339540)\
[Medium article 1](https://medium.com/starschema-blog/monitor-your-infrastructure-with-influxdb-and-grafana-on-kubernetes-a299a0afe3d2)\
[Medium article 2](https://www.gojek.io/blog/diy-set-up-telegraf-influxdb-grafana-on-kubernetes)\
[InfluxDB Documentation](https://docs.influxdata.com/)\
[Telegraf Documentation](https://www.influxdata.com/time-series-platform/telegraf/)\


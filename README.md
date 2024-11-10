# Задание 1

# Разрорачивание прометеуса:

![Иллюстрация к проекту](https://github.com/randnull/sre-hw/blob/main/photo/docker-prom.png)

# Базовые метрики и ui:

![Иллюстрация к проекту](https://github.com/randnull/sre-hw/blob/main/photo/prom-ui.png)
![Иллюстрация к проекту](https://github.com/randnull/sre-hw/blob/main/photo/somemetric.png)
![Иллюстрация к проекту](https://github.com/randnull/sre-hw/blob/main/photo/some2metric.png)

# Задание 2

Изменения в Oncall + манифестах для добавления метрик:

Dockerfile: (&& pip install prometheus-client)
```
RUN chown -R oncall:oncall /home/oncall/source /var/log/nginx /var/lib/nginx \
   && sudo -Hu oncall mkdir -p /home/oncall/var/log/uwsgi /home/oncall/var/log/nginx /home/oncall/var/run /home/oncall/var/relay \
   && sudo -Hu oncall python3 -m venv /home/oncall/env \
   && sudo -Hu oncall /bin/bash -c 'source /home/oncall/env/bin/activate && cd /home/oncall/source && pip install wheel && pip install . && pip install prometheus-client'
```
docker-compose: (- "8000:8000")
```
services:
 oncall-web:
   build: .
   hostname: oncall
   ports:
     - "8080:8080"
     - "8000:8000"
   environment:
```
config.yaml:

```
apiVersion: v1
kind: ConfigMap
metadata:
 name: oncall
data:
 oncall.conf: |
   ---
   server:
     host: 0.0.0.0
     port: 8080
   debug: True
   oncall_host: oncall.local
   metrics: prometheus
   prometheus:
     oncall-notifier:
       server_port: 8000
   db:
```



deployment.yaml: (- containerPort: 8000)

```
spec:
 replicas: 2
 selector:
   matchLabels:
     app.kubernetes.io/name: oncall
     app.kubernetes.io/component: web
 template:
   metadata:
     labels:
       app.kubernetes.io/name: oncall
       app.kubernetes.io/component: web
   spec:
     containers:
     - name: oncall
       image: oncall:latest
       env:
       - name: DOCKER_DB_BOOTSTRAP
         value: "1"
       imagePullPolicy: Never
       ports:
       - containerPort: 8080
       - containerPort: 8000
       livenessProbe:
         httpGet:
           path: /healthcheck
           port: 8080
```
ingress.yaml:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
 name: oncall-ingress
 annotations:
   nginx.ingress.kubernetes.io/rewrite-target: /$1
   nginx.ingress.kubernetes.io/use-regex: "true"
spec:
 rules:
   - host: oncall.local
     http:
       paths:
         - path: "/(.*)"
           pathType: Prefix
           backend:
             service:
               name: oncall
               port:
                 number: 8080
         - path: /metrics
           pathType: Prefix
           backend:
             service:
               name: oncall
               port:
                 number: 8000
```


service.yaml:

```

apiVersion: v1
kind: Service
metadata:
 name: oncall
spec:
 type: NodePort
 selector:
   app.kubernetes.io/name: oncall
 ports:
   - name: http
     protocol: TCP
     port: 8080
     targetPort: 8080
   - name: metrics
     protocol: TCP
     port: 8000
     targetPort: 8000
```

Метрики по адресу: oncall.local/metrics
![Иллюстрация к проекту](https://github.com/randnull/sre-hw/blob/main/photo/oncall-metrics.png)



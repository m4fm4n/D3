D3.4.

###Задание:

1. Создать сущность Deployment c тремя репликами веб-сервера;
        
2. Добавить к нему сервис;
        
3. Интегрировать его внутрь конфигурации с помощью ConfigMap;
        
4. Добавить секрет для создания basic auth аутентификации в nginx.

---

###Выполнение:

Был создан файл-манифест depoy-nginx.yaml со следующими свойствами:

1. kind: Deployment

- образ - nginx:1.21.1-alpine

- имя - nginx-sf

- количество реплик - 3

- путь до файла конфигурации в Pod-е - /etc/nginx/nginx.conf


2. kind: Service

- имя сервиса sf-webserver

- внешний порт - 80


3. kind: ConfigMap

```
data:
  nginx.conf: |
    user nginx;
    worker_processes 1;
    events {
      worker_connections 10240;
    }
    http {
      server {
        listen       80;
        server_name  localhost;
        location / {
          root   /usr/share/nginx/html;
          index  index.html index.htm;
        }
      }
    }
```

---

После команды применения 'kubectl apply -f deploy-nginx.yaml' была произведена проверка работоспособности:

kubectl get deploy  #  отобразить deploy в текущем контексте и namespace.

kubectl get pods  #  отобразить Pod-ы 

Затем через команду `kubectl exec -it <nameOfPod> -- sh` было осуществлено подключение к терминалу одного из подов. Через команду 'cat /etc/nginx/nginx.conf' была проверена конфигурация nginx.

Проверка поднятия Service выполнялась через команду 'kubectl describe service sf-webserver'


---

4. kind: Secret

Далее для аутентификации в nginx были подготовлены данные для отправки в Pod-ы:


`htpasswd -c myAuth user1`

`cat myAuth | base64`


Был создан файл secret.yaml с данными для передачи:

```
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth
type: Opaque
data:
  user1: dXNlcjE6JGFwcjEkMDJnaGpGSFkkeHB6QkJidW05ejZ0U3k4ZmVSaFFRLgo=
```

В файл deploy-nginx.yaml, в разделе Deployment,  добавлены следующие строки:

```
...
spec:
  ...
  template:
    ...
    spec:
      containers:
      ...
      volumeMounts:
      ...
      - name: auth
        mountPath: "/etc/nginx/conf"
        readOnly: true
      volumes:
      ...
      - name: auth
        secret:
          secretName: basic-auth
          items:
          - key: user1
          path: htpasswd
```

В разделе ConfigMap изменено:

```
data:
  nginx.conf: |
    user nginx;
    worker_processes 1;
    events {
      worker_connections 10240;
    }
    http {
      server {
        listen       80;
        server_name  localhost;
        location / {
          root   /usr/share/nginx/html;
          index  index.html index.htm;
        }
        auth_basic "User Auth";
        auth_basic_user_file conf/htpasswd;
      }
    }
```

---

Применение новой кофигурации через команду 'kubectl apply -f .'

Далее проверка через команды:


`kubectl get pods`

`kubectl exec -it <nameOfPod> -- curl sf-webserver`

`kubectl exec -it <nameOfPod> -- curl -u user1:password1 sf-webserver`


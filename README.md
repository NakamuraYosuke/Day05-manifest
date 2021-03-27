# Day05-manifest

## 事前作業
crcを起動後、crc環境変数を読み込みます。
```
$ eval $(crc oc-env)
```

developerユーザでログインします。
```
$ oc login -u developer -p developer https://api.crc.testing:6443
```

Dockerアプリケーションを起動しておきます。
```
$ open /Applications/Docker.app
```

## Dockerfileを用いたアプリケーションのビルド

以下、例では作業ディレクトリを`~/container-build`とします。

```
$ mkdir -p ~/container-build
```

下記の内容のDockerfileを作業ディレクトリに作成します。
```
$ vim ~/container-build/Dockerfile
```

```
FROM nakanakau/httpd-parent

# Change the port to 8080
EXPOSE 8080

# Labels consumed by OpenShift
LABEL io.k8s.description="A basic Apache HTTP Server child image, uses ONBUILD" \
      io.k8s.display-name="Apache HTTP Server" \
      io.openshift.expose-services="8080:http" \
      io.openshift.tags="apache, httpd"

# Change web server port to 8080
RUN sed -i "s/Listen 80/Listen 8080/g" /etc/httpd/conf/httpd.conf

# Permissions to allow container to run on OpenShift
RUN chgrp -R 0 /var/log/httpd /var/run/httpd && \
    chmod -R g=u /var/log/httpd /var/run/httpd

# Run as a non-privileged user
USER 1001
```

次に、作業ディレクトリに`src`という名前のディレクトリを作成し、そこに下記のHTMLファイルを`index.html`という名前で作成します。

```
$ mkdir -p ~/container-build/src
```

```
$ vim index.html
```
```
<!DOCTYPE html>
<html>
<body>
  Hello from the Apache child container!
</body>
</html>
```

crcに`container-build`という名前のnamespaceを作成して切り替えます。
```
$ kubectl create ns container-build
$ kubectl config set-context $(kubectl config current-context) --namespace=container-build
```

先ほどのDockerfileを`hola`という名前でビルドします。
```
$ cd ~/container-build
$ docker build -t hola .
```

イメージを確認します。
```
$ docker images
```

Docker Hubにログインします。
```
$ docker login
```

Docker Hubに上記holaイメージをpushします。
```
$ docker tag hola <自身のアカウント>/hola:latest
$ docker push <自身のアカウント>/hola:latest
```

次にDeploymentとServiceを作成します。
```
$ vim ~/container-build/deployment.yaml
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hola
  name: hola
  namespace: container-build
spec:
  replicas: 1
  selector:
    matchLabels:
      deployment: hola
  template:
    metadata:
      labels:
        deployment: hola
    spec:
      containers:
      - image: <自身のアカウント>/hola:latest
        name: hola
        ports:
        - containerPort: 8080
          protocol: TCP
```

```
$ vim ~/container-build/service.yaml
```
```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: hola
  name: hola
  namespace: container-build
spec:
  ports:
  - name: 8080-tcp
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    deployment: hola
  type: NodePort
```

DeploymentとServiceを適用します。
```
$ kubectl apply -f ~/container-build/deployment.yaml
$ kubectl apply -f ~/container-build/service.yaml
```

NodePortで作られたServiceのポート番号を確認します。
```
$ kubectl get svc hola
NAME   TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
hola   NodePort   10.217.5.43   <none>        8080:30696/TCP   5m39s
```

crcのIPアドレスを用いてアクセスしてみます。
```
$ crc ip
192.168.64.2
```

正常にデプロイされているかを確認します。
```
$ curl -s http://192.168.64.2:30696
```

下記のメッセージが出力されれば成功です。
```
<!DOCTYPE html>
<html>
<body>
  Hello from the Apache child container!
</body>
</html>
```

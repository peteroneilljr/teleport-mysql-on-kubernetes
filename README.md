## Demo Self Hosted Mysql DB and Teleport Agent on Kubernetes

### Create namespace

```sh
kubectl create ns tel-mysql
```

### Create Teleport Certificates
```sh
HOSTNAME=mysql.tel-mysql.svc.cluster.local
KEYNAME=mysql-teleport
NAMESPACE=tel-mysql
tctl auth sign --format=db --host=$HOSTNAME --out=./certs/$KEYNAME --ttl=2190h
kubectl create secret generic mysql-certs  -n $NAMESPACE --from-file=./certs/$KEYNAME.crt --from-file=./certs/$KEYNAME.key --from-file=./certs/$KEYNAME.cas
```

### Create Mysql Credentials
```sh
NAMESPACE=tel-mysql
kubectl -n $NAMESPACE create secret generic mysql-vars \
    --from-literal=MYSQL_USER="mysql" \
    --from-literal=MYSQL_PASSWORD="$(openssl rand -base64 24)" \
    --from-literal=MYSQL_ROOT_PASSWORD="$(openssl rand -base64 24)"
```

### Deploy Mysql
```sh
NAMESPACE=tel-mysql
kubectl apply -f deploy-mysql.yaml -n $NAMESPACE
```

### Create values, token is valid for 1 hour
```sh
PROXY="example.teleport.sh:443"
TOKEN="$(tctl tokens add --type=db --format=text)"
DB_NAME="k8s-mysql"
DB_URI="mysql.tel-mysql.svc.cluster.local:3306"

cat > values.yaml <<EOF
roles: db
proxyAddr: $PROXY
authToken: $TOKEN
databases:
  - name: $DB_NAME
    uri: $DB_URI
    protocol: mysql
    static_labels:
      env: dev
      deploy: k8s
EOF
```

### Get Teleport Agent Helm Chart
```sh
helm repo add teleport https://charts.releases.teleport.dev
helm repo update
```

### Install Teleport Agent
```sh
NAMESPACE=tel-mysql
CHART_NAME="tel-mysql-agent"
helm install $CHART_NAME teleport/teleport-kube-agent \
  --namespace $NAMESPACE \
  --version 16.4.10 \
  -f values.yaml
```

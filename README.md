# Getting started with SpiceDB Operator model

## Instal SpiceDB Operator

```bash
kubectl apply --server-side -k github.com/authzed/spicedb-operator/config
```

## Install mysql

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install mysql bitnami/mysql -f mysql-values.yaml
```

```bash
echo Username: root
MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default mysql -o jsonpath="{.data.mysql-root-password}" | base64 -d)

kubectl run mysql-client --rm --tty -i --restart='Never' --image  docker.io/bitnami/mysql:8.0.30-debian-11-r6 --namespace default --env MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASSWORD --command -- mysql -h mysql.default.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD" -e 'CREATE DATABASE IF NOT EXISTS spicedb'
```

## Create SpiceDBCluster

```bash
kubectl apply --server-side -f - <<EOF
apiVersion: authzed.com/v1alpha1
kind: SpiceDBCluster
metadata:
  name: dev
spec:
  config:
    image: ghcr.io/authzed/spicedb:v1.13.0
    replicas: 2
    datastoreEngine: mysql
    logLevel: debug

  secretName: dev-spicedb-config
---
apiVersion: v1
kind: Secret
metadata:
  name: dev-spicedb-config
stringData:
  datastore_uri: "root:$MYSQL_ROOT_PASSWORD@tcp(mysql.default.svc.cluster.local:3306)/spicedb?charset=utf8mb4,utf8"
  preshared_key: "averysecretpresharedkey"
EOF
```

```bash
kubectl port-forward deployment/dev-spicedb 50051:50051
```

## Connect and Verify

```bash
brew install zed
```

```bash
zed context set local localhost:50051 "averysecretpresharedkey" --insecure

zed schema write <(cat << EOF
definition blog/user {}

definition blog/post {
	relation reader: blog/user
	relation writer: blog/user

	permission read = reader + writer
	permission write = writer
}
EOF
)

zed schema read --insecure
```

# Kubernetes - Galera Cluster

This repository stores the files needed to deploy a **galera cluster** on Kubernetes, more specifically on [DigitalOcean](https://www.digitalocean.com/).

## Steps

First we need the *.cnf* with the [wsrep](https://galeracluster.com/library/documentation/mysql-wsrep-options.html) options:

**galera.cnf**

```ini
[galera]
wsrep_on=ON
wsrep_provider=/usr/lib/libgalera_smm.so
wsrep_cluster_name=dragonfly
wsrep_cluster_address=gcomm://mariadb-0.mariadb.mariadb:4567,mariadb-1.mariadb.mariadb:4567,mariadb-2.mariadb.mariadb:4567
binlog_format=row
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
```

Then, add this file as a **ConfigMap** on the cluster:

  kubectl create cm galera-cnf --from-file galera.cnf

Start only the **first** server, with this modification on **mariadb.yml** file:

```yaml
# omitted
  replicas: 1
# omitted
      containers:
      - name: mariadb
        image: mariadb:10.4
        args:
        - --wsrep-new-cluster
        env:
        - name: MYSQL_ROOT_PASSWORD
# omitted
```

Wait until the container finish the bootstrap process. You can follow with the **logs** subcommand:

```bash
kubectl logs -f mariadb-0
```

Once finished, remove the `--wsrep-new-cluster` with the `args` key, and raise the `replicas` to 3.

## Testing

Once the **galera cluster** finished the startup, you can access from inside the cluster on **mariadb** service or in one of the following DNS registers:

- mariadb-0.mariadb
- mariadb-1.mariadb
- mariadb-2.mariadb

There is a lot of ways to test, this is just one of them.

```bash
kubectl create deploy connector --image alpine
kubectl patch deploy connector -p '{"spec" : {"template" : {"spec" : {"containers" : [{"name" : "alpine", "tty" : true, "stdin" : true}]}}}}'
kubectl get pods
```

Install the **mysql client** from inside the container:

```bash
apk add mysql-client
```

Generate a random user name list - single column - on https://mockaroo.com/ and stores inside the **connector** container, for example, in a file named *users.txt*.

```bash
mysql -h mariadb -u user -p123 base -e "CREATE TABLE usuarios (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(50) DEFAULT '')"
cat users.txt | while read X; do mysql -h mariadb -u user -p123 base -e "INSERT INTO users (name) VALUES ('$X')"; done
```

# Creation of CustomResourceDefinitions

```
$ kubectl create namespace meetup
$ kubectl create -f databases-crd.yaml
```

# Watching CustomResourceDefinitions

```
$ http --json "http://localhost:8080/apis/meetup.example.com/v1/namespaces/meetup/databases/wordpress?watch=true
```

# CRUD of a CustomResource

```
$ kubectl create -f wordpress-database.yaml
$ kubectl get databases
$ kubectl delete databases wordpress
```

# Using a Finalizer

```
$ kubectl create -f wordpress-database.yaml
$ kubectl edit database wordpress

# add metadata.finalizer: ["meetup.example.com/deleteMysqlDB"]

$ kubectl delete database wordpress
$ kubectl get database wordpress -o yaml

$ kubectl edit database wordpress

# remove metadata.finalizer

$ kubectl get database wordpress -o yaml
```

# Create init.databases.meetup.example.com Initializer

```
$ kubectl create -f databases-initializerconfiguration.yaml
```

In another terminal:

```
$ kubectl get databases
$ http --json "http://localhost:8080/apis/meetup.example.com/v1/namespaces/meetup/databases/wordpress
```

Show uninitialized resources:

```
$ http --json "http://localhost:8080/apis/meetup.example.com/v1/namespaces/meetup/databases/wordpress?includeUninitialized=true
```

Delete the initializer to accept the creation. Be fast, 60 seconds timeout! If you are too slow the initializer is ignored, compare failure mode in initializer spec.

```
http --json "http://localhost:8080/apis/meetup.example.com/v1/namespaces/meetup/databases/wordpress?includeUninitialized=true" |
   jq 'del(.metadata.initializers)' | 
   jq '.metadata.finalizers=["deleteMysqlDB"]' | 
   http PUT "http://localhost:8080/apis/meetup.example.com/v1/namespaces/meetup/databases/wordpress?includeUninitialized=true"
```

Delete again:

```
kubectl delete database wordpress
```

And now reject the request:

```
$ kubectl create -f databases-initializerconfiguration.yaml
```

Again in another terminal:

```
http --json "http://localhost:8080/apis/meetup.example.com/v1/namespaces/meetup/databases/wordpress?includeUninitialized=true" | 
   jq 'del(.metadata.initializers.pending)' | 
   jq '.metadata.initializers.result={"status":"Failure", "code":401, "reason":"Rejected by meetup"}' | 
   http PUT "http://localhost:8080/apis/meetup.example.com/v1/namespaces/meetup/databases/wordpress?includeUninitialized=true"
```

Of course, it's not good practice to delete the whole `.metadata.initializers.pending` value. With a bit more jq magic (`Del(.Metadata.Initializers.pending[] | select(.name="init.databases.meetup.example.com"))`) or proper implementation in Go, this can easily be done.


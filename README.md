# Quickstart - Vert.x

Quickstart where the Vert.x container retrieves the Application parameters using a Kubernetes ConfigMap. For this quickstart, we will instantiate a Vertx HTTP Server
where the port number has been defined using a Kubernetes configMap using this name `app-config`. 

```java
 ConfigurationStoreOptions appStore = new ConfigurationStoreOptions();
 appStore.setType("configmap")
         .setConfig(new JsonObject()
                 .put("namespace", "vertx-demo")
                 .put("name", "app-config")); / Name of the ConfigMap to be fetched 

 configurationservice = ConfigurationService.create(vertx, new ConfigurationServiceOptions()
         .addStore(appStore));

 Router router = Router.router(vertx);

 router.route().handler(BodyHandler.create());
 router.get("/products/:productID").handler(this::handleGetProduct);
 router.put("/products/:productID").handler(this::handleAddProduct);
 router.get("/products").handler(this::handleListProducts);

 configurationservice.getConfiguration(ar -> {
     // We can retrieve the port key from the cache created when the Configuration Service has retrieve the values from Kubernetes
     int port = configurationservice.getCachedConfiguration().getInteger("port") != null ? configurationservice.getCachedConfiguration().getInteger("port") : 8080;
     LOGGER.info("Config Map - port : " + port);
     vertx.createHttpServer().requestHandler(router::accept).listen(port);
 });
```

The config map contains under the `app.json` key, the list of the key/value pairs defined 
using JSon as Dataformat for our application as you can see hereafter :

```yaml
apiVersion: v1
data:
  app.json: |-
    {
        "logging":"debug",
        "hostname":"127.0.0.1",
        "port":"8080"
    }
kind: ConfigMap
metadata:
  creationTimestamp: 2016-09-14T12:13:08Z
  name: app-config
  namespace: vertx-demo
  resourceVersion: "6232"
  selfLink: /api/v1/namespaces/vertx-demo/configmaps/app-config
  uid: 990dd236-7a74-11e6-8d26-868f517b6834
```


# Instructions

* Launch minishift

```
minishift delete
minishift start --deploy-registry=true --deploy-router=true --memory=4048 --vm-driver="xhyve"
minishift docker-env
eval $(minishift docker-env)
```

* Setup the Kubernetes ENV variable required on your machine 

```
export KUBERNETES_MASTER=https://$(minishift ip):8443
``` 
   
* Log on to openshift
```    
oc login $(minishift ip):8443 -u admin -p admin
```    
# Create a new project

```    
oc new-project vertx-demo
oc policy add-role-to-user view openshift-dev -n vertx-demo
oc policy add-role-to-group view system:serviceaccounts -n vertx-demo
```

# Create the ConfigMap

```
oc create configmap game-config --from-file=src/main/resources/game.properties
oc create configmap ui-config --from-file=src/main/resources/ui.json
oc create configmap app-config --from-file=src/main/resources/app.json
```

# To consult the configMap (optional)

```
oc get configmap/game-config -o yaml
oc get configmap/ui-config -o yaml
oc get configmap/app-config -o yaml
```

* Build and deploy the project
   
```
mvn -Popenshift   
```

# Consult the Service deployed 

First, we must retrieve the IP Address of the service exposed by the OpenShift Router to our host machine

```
minishift service simple-vertx-configmap -n vertx-demo --url=true
http://192.168.64.12:32741
```

Next, we can use curl or httpie tool to fetch the products from the REST service

```
curl http://192.168.64.12:32741/products
HTTP/1.1 200 OK
Content-Length: 252
content-type: application/json

[
    {
        "id": "prod7340",
        "name": "Tea Cosy",
        "price": 5.99,
        "weight": 100
    },
    {
        "id": "prod3568",
        "name": "Egg Whisk",
        "price": 3.99,
        "weight": 150
    },
    {
        "id": "prod8643",
        "name": "Spatula",
        "price": 1.0,
        "weight": 80
    }
]
```

We can also grab the info about one product

```
curl http://192.168.64.12:32741/products/prod7340
HTTP/1.1 200 OK
Content-Length: 82
content-type: application/json

{
    "id": "prod7340",
    "name": "Tea Cosy",
    "price": 5.99,
    "weight": 100
}
```

or create a new product

```
curl -X PUT http://192.168.64.12:32741/products/prod1122 -d @src/main/resources/prod1122.json
```


# Delete Replication controller, service, ConfigMap

```
oc delete service simple-vertx-configmap
oc delete rc simple-config-map

oc delete configmap/game-config
oc delete configmap/ui-config
oc delete configmap/app-config
```

# Resize Pods

```
oc scale rc simple-config-map --replicas=0
```

# Not used

```
Json
oc create configmap game-config --from-file=src/main/resources/game.json
oc create configmap ui-config --from-file=src/main/resources/ui.json
oc create configmap app-config --from-file=src/main/resources/app.json

Properties
oc create configmap game-config --from-file=src/main/resources/game.properties
oc create configmap ui-config --from-file=src/main/resources/ui.properties
oc create configmap app-config --from-file=src/main/resources/app.properties
```

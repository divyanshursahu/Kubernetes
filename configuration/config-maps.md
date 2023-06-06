---
description: >-
  https://matthewpalmer.net/kubernetes-app-developer/articles/ultimate-configmap-guide-kubernetes.html
---

# Config maps

A ConfigMap is an API object used to store non-confidential data in key-value pairs. [Pods](https://kubernetes.io/docs/concepts/workloads/pods/) can consume ConfigMaps as environment variables, command-line arguments, or as configuration files in a [volume](https://kubernetes.io/docs/concepts/storage/volumes/).

A ConfigMap allows you to decouple environment-specific configuration from your [container images](https://kubernetes.io/docs/reference/glossary/?all=true#term-image), so that your applications are easily portable.

<figure><img src="../.gitbook/assets/image (1).png" alt="" width="327"><figcaption></figcaption></figure>

potential drawback of ConfigMaps is that files must be limited to 1MB. Larger datasets may require different storage methods, such as separate file mounts, file services or databases.

### ConfigMap object <a href="#configmap-object" id="configmap-object"></a>

A ConfigMap is an [API object](https://kubernetes.io/docs/concepts/overview/working-with-objects/#kubernetes-objects) that lets you store configuration for other objects to use. Unlike most Kubernetes objects that have a `spec`, a ConfigMap has `data` and `binaryData` fields. These fields accept key-value pairs as their values.&#x20;

The name of a ConfigMap must be a valid [DNS subdomain name](https://kubernetes.io/docs/concepts/overview/working-with-objects/names#dns-subdomain-names).

Each key under the `data` or the `binaryData` field must consist of alphanumeric characters, `-`, `_` or `.`. The keys stored in `data` must not overlap with the keys in the `binaryData` field.



### Create a configmap:

You can create config maps from directories, files, or literal values using **kubectl create configmap**.

```
$ cat app.properties 
environment=production
logging=INFO
logs_path=$APP_HOME/logs/
parllel_jobs=3
wait_time=30sec
```

and

```
kubectl create configmap app-config \ 
  --from-file configs/app.properties
configmap "app-config" created
```

which is the same as

```
kubectl create configmap app-config \ 
  --from-file configs/
```

and

```
kubectl create configmap app-config \ 
  --from-literal environment=production \
  --from-literal logging=INFO
  .......
```

You can create ConfigMaps based on one file, several files, directories, or env-files (lists of environment variables). The basename of each file is used as the key, and the contents of the file becomes the value.

|                           |                                                                                                                                                |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **ConfigMap Data Source** | **Example kubectl command**                                                                                                                    |
| **Single file**           | `kubectl create configmap app-settings --from-file=app-container/settings/app.properties`                                                      |
| **Multiple files**        | `kubectl create configmap app-settings --from-file=app-container/settings/app.properties--from-file=app-container/settings/backend.properties` |
| **Env-file**              | `kubectl create configmap app-env-file--from-env-file=app-container/settings/app-env-file.properties`                                          |
| **Directory**             | `kubectl create configmap app-settings --from-file=app-container/settings/`                                                                    |

List configmap:

```
kubectl get configmap/app-config
NAME         DATA      AGE
app-config   1         1m
```

View configmap:

```
kubectl describe configmap/app-config
Name:         app-config
Namespace:    default
Labels:       < none >
Annotations:  < none >
Data
====
app.properties:
----
environment=production
logging=INFO
logs_path=$APP_HOME/logs/
parllel_jobs=3
wait_time=30sec
Events:  < none >
```

To view the ConfigMap in YAML format, use this command:

```
kubectl get configmaps <name> -o yaml
```

### ConfigMaps and Pods <a href="#configmaps-and-pods" id="configmaps-and-pods"></a>

You can write a Pod `spec` that refers to a ConfigMap and configures the container(s) in that Pod based on the data in the ConfigMap. The Pod and the ConfigMap must be in the same [namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces).

Here's an example ConfigMap that has some keys with single values, and other keys where the value looks like a fragment of a configuration format.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # property-like keys; each key maps to a simple value
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"

  # file-like keys
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5    
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true 
```

There are four different ways that you can use a ConfigMap to configure a container inside a Pod:

1. Inside a container command and args
2. Environment variables for a container
3. Add a file in read-only volume, for the application to read
4. Write code to run inside the Pod that uses the Kubernetes API to read a ConfigMap

Here's an example Pod that uses values from `game-demo` to configure a Pod

{% code title="configmap/configure-pod.yaml " %}
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: alpine
      command: ["sleep", "3600"]
      env:
        # Define the environment variable
        - name: PLAYER_INITIAL_LIVES # Notice that the case is different here
                                     # from the key name in the ConfigMap.
          valueFrom:
            configMapKeyRef:
              name: game-demo           # The ConfigMap this value comes from.
              key: player_initial_lives # The key to fetch.
        - name: UI_PROPERTIES_FILE_NAME
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: ui_properties_file_name
      volumeMounts:
      - name: config
        mountPath: "/config"
        readOnly: true
  volumes:
  # You set volumes at the Pod level, then mount them into containers inside that Pod
  - name: config
    configMap:
      # Provide the name of the ConfigMap you want to mount.
      name: game-demo
      # An array of keys from the ConfigMap to create as files
      items:
      - key: "game.properties"
        path: "game.properties"
      - key: "user-interface.properties"
        path: "user-interface.properties"
```
{% endcode %}

A ConfigMap doesn't differentiate between single line property values and multi-line file-like values. What matters is how Pods and other objects consume those values.

For this example, defining a volume and mounting it inside the `demo` container as `/config` creates two files, `/config/game.properties` and `/config/user-interface.properties`, even though there are four keys in the ConfigMap. This is because the Pod definition specifies an `items` array in the `volumes` section. If you omit the `items` array entirely, every key in the ConfigMap becomes a file with the same name as the key, and you get 4 files.

### Using ConfigMaps <a href="#using-configmaps" id="using-configmaps"></a>

ConfigMaps can be mounted as data volumes. ConfigMaps can also be used by other parts of the system, without being directly exposed to the Pod. For example, ConfigMaps can hold data that other parts of the system should use for configuration.

The most common way to use ConfigMaps is to configure settings for containers running in a Pod in the same namespace. You can also use a ConfigMap separately.

For example, you might encounter [addons](https://kubernetes.io/docs/concepts/cluster-administration/addons/) or [operators](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) that adjust their behavior based on a ConfigMap.

#### Using ConfigMaps as files from a Pod <a href="#using-configmaps-as-files-from-a-pod" id="using-configmaps-as-files-from-a-pod"></a>

To consume a ConfigMap in a volume in a Pod:

1. Create a ConfigMap or use an existing one. Multiple Pods can reference the same ConfigMap.
2. Modify your Pod definition to add a volume under `.spec.volumes[]`. Name the volume anything, and have a `.spec.volumes[].configMap.name` field set to reference your ConfigMap object.
3. Add a `.spec.containers[].volumeMounts[]` to each container that needs the ConfigMap. Specify `.spec.containers[].volumeMounts[].readOnly = true` and `.spec.containers[].volumeMounts[].mountPath` to an unused directory name where you would like the ConfigMap to appear.
4. Modify your image or command line so that the program looks for files in that directory. Each key in the ConfigMap `data` map becomes the filename under `mountPath`.

This is an example of a Pod that mounts a ConfigMap in a volume:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    configMap:
      name: myconfigmap
```

Each ConfigMap you want to use needs to be referred to in `.spec.volumes`.

If there are multiple containers in the Pod, then each container needs its own `volumeMounts` block, but only one `.spec.volumes` is needed per ConfigMap

### References:

{% embed url="https://kubernetes.io/docs/concepts/configuration/configmap/" fullWidth="false" %}

{% embed url="https://www.aquasec.com/cloud-native-academy/kubernetes-101/kubernetes-configmap/" %}

{% embed url="https://matthewpalmer.net/kubernetes-app-developer/articles/ultimate-configmap-guide-kubernetes.html" %}

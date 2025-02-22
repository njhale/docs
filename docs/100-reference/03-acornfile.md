---
title: Acornfile
---

## Root

The root level elements are,
[args](#args),
[services](#services),
[acorns](#acorns),
[containers](#containers),
[jobs](#jobs),
[routers](#routers),
[volumes](#volumes),
[secrets](#secrets),
and [localData](#localdata).

[containers](#containers),
[jobs](#jobs),
and [routers](#routers)
are all maps where the keys must be unique across all types. For example, it is
not possible to have a container named `foo` and a job named `foo`, they will conflict and fail. Additional
the keys could be using in a DNS name so the keys must only contain the characters `a-z`, `0-9` and `-`.

```acorn
// User configurable values that can be changed at build or run time.
args: {
}

// Definition of services the Acorn app will consume
services: {
}

// Definition of Acorns that will consumed by this Acorn app
acorns: {
}

// Definition of containers to run
containers: {
}

// Defintion of jobs to run
jobs: {
}

// Defintion of HTTP routes to services
routers: {
}

// Definition of volumes that this acorn needs to run
volumes: {
}

// Definition of secrets that this acorn needs to run
secrets: {
}

// Arbitrary information that can be embedded to help render this Acornfile
localData: {
}
```

## containers

`containers` defines the templates of containers to be ran. Depending on the
scale parameter 1 or more containers can be created from each template (including their [sidecars](#sidecars)).

```acorn
containers: web: {
 image: "nginx"
 ports: publish: "80/http"
}
```

### dirs, directories

`dirs` configures one or more volumes to be mounted to the specified folder.  The `dirs` parameter is a
map structure where the key is the folder name in the container and the value is the referenced volume. Refer
to the [volumes](#volumes) section for more information on volume types.

```acorn
containers: default: {
 image: "nginx"
 dirs: {
  // A volume named "volume-name" will be mounted at /var/tmp
  "/var/tmp": "volume-name"
  
  // A volume named "volume-name" will be mounted at /var/tmp-full with the size of 20G and an
  // access mode of readWriteMany
  "/var/tmp-full": "volume://volume-name?size=20G,accessMode=readWriteMany"
  
  // An ephemeral volume will be created and mounted to "/var/tmp-ephemeral"
  "/var/tmp-ephemeral": "ephemeral://"
  
  // An ephemeral volume named "shared" will be created and mounted to "/var/tmp-ephemeral"
  "/var/tmp-ephemeral-named": "ephemeral://shared"
  
  // A folder will be created at /var/tmp-secret/ where the filenames are the
  // key names of the secret "sec-name" and the contents of each file is the corresponding
  // secret data value
  "/var/tmp-secret": "secret://sec-nam"
  
  // The local folder ./www will be copied during build into the container image
  // as /var/www.  If running in dev mode the directory will be syncronized live with
  // changes.  Local folders must start with "./".
  "/var/www": "./www"
 }
 sidecars: sidecar: {
  image: "ubuntu"
  dirs: {
   // An ephemeral volume named "shared" will be mounted to /var/tmp with the contents of
   // the volume shared with the main containers /var/tmp-ephemeral-named folder
            "/var/tmp": "ephemeral://shared"
  }
 }
```

### files

`files` will create files in the container with content from the Acornfile or the value of a secret. The
`files` parameter is a map structure with the key being the file name and the value being the text of the file
or a reference to a secret value. The default mode for files is `0644` unless the file ends with `.sh` or contains
`/bin/` or `/sbin/`.  In those situations the mode will be `0755`.

```acorn
containers: default: {
 image: "nginx"
 files: {
  // A file named /var/tmp/file.txt will be created with mode 0644
  "/var/tmp/file.txt": "some file contents"
  
  // A file named /run/secret/password will be created with mode 0400 with the
  // contents of from the secret named "sec-name" and the value of the data
  // key named "key".
  "/run/secret/password": "secret://sec-name/key?mode=0400"
  
  // By default if a secret value changes the application will be restarted.
  // the following example will cause the container to not be restarted when
  // the secret value changes, but instead the container is dynamically updated
  "/run/secret/password-reload": "secret://sec-name/key?onchange=no-action"
  
  // A file /var/tmp/other.txt will be created with a custom mode value "0600"
  "/var/tmp/other.txt": {
   content: "file content"
   mode: "0600"
  }
  
 }
}
```

### image

`image` refers to the OCI (Docker) image to run for this container.

```acorn
containers: web: {
 image: "nginx"
}
```

### build

`build` contains the information need to build an OCI image to run for this container

```acorn
containers: build1: {
 // Build the Docker image using the context dir "." and the "./Dockerfile".
 build: "."
}

containers: build2: {
 build: {
  // Build using the context dir "./subdir"
  context: "./subdir"
  // Build using the "./subdir/Dockerfile"
  dockerfile: "./subdir/Dockerfile"
  // Build with the multi-stage target named "multistage-target"
  target: "multistage-target"
  // Pass the following build arguements to the dockerfile
  buildArgs: {
   "arg1": "value1"
   "arg2": "value2"
  }
 }
}
```

### command, cmd

`command` will overwrite the `CMD` value set in the Dockerfile for the running container

```acorn
containers: arg1: {
 image: "nginx"
 // This command will be parsed as a shell expression and turned into an array and ran
 cmd: #"/bin/sh -c "echo hi""#
}

containers: arg2: {
 image: "nginx"
 // The following will not be parsed and will be ran as defined.
 cmd: ["/bin/sh", "-c", "echo hi"]
}

```

### entrypoint

`entrypoint` will overwrite the `ENTRYPOINT` value set in the Dockerfile for this running container

```acorn
containers: arg1: {
 image: "nginx"
 // This command will be parsed as a shell expression and turned into an array and ran
 entrypoint: #"/bin/sh -c "echo hi""#
}

containers: arg2: {
 image: "nginx"
 // The following will not be parsed and will be ran as defined.
 entrypoint: ["/bin/sh", "-c", "echo hi"]
}
```

### env, environment

`env` will set environment variables on the defined container.  The value of the environment variable
may be static text or a value from a secret.

```acorn
containers: env1: {
 image: "nginx"
 env: [
     // An environment variable of name "NAME" and value "VALUE" will be set
  "NAME=VALUE",

     // An environment variable of name "SECRET" and value of the key "key" in the
     // secret named "sec-name" will be set. When this secret changes the container
     // will not be restarted.
  "SECRET=secret://sec-name/key?onchange=no-action"
 ]
}

containers: env1: {
 image: "nginx"
 // The same configuration as above but in map form
 env: [
  NAME: "VALUE"
  SECRET: "secret://sec-name/key?onchange=no-action"
 ]
}
```

### workDir, workingDir

`workDir` sets the current working directory of the running process defined in `cmd` and `entrypoint`

```acorn
containers: env1: {
 image: "nginx"
 command: "ls"
 // Run the command "ls", as defined above, in the directory "/tmp"
 workDir: "/tmp"
}
```

### consumes

`consumes` should always be used when a service is required for the container to run. The `consumes` field defines both the dependency and permissions that the container will need to interact with the service. When no permissions are defined on the service, the behavior is the same as `dependsOn`.

```acorn
services: db: {
    image: "ghcr.io/acorn-io/aws/rds/aurora/mysql/cluster:*"
}

containers: app: {
    image: "nginx"
    consumes: ["db"]
    // ...
}
```

### dependsOn

`dependsOn` is used to enforce ordering on startup by preventing a container from being created and/or updated until all dependencies are considered ready. Dependencies are considered ready as soon as the [ready probe](#probes-probe) passes. If there is no ready probe, the dependency is considered ready when the container starts.

```acorn
containers: web: {
 image: "nginx"
 dependsOn: ["db"]
}
containers: db: {
 // ...
 image: "mariadb"
}
```

### ports

`ports` defines which ports are available on the container and the default level of access. Ports
are defined with three different access modes: internal, expose, publish. Internal ports are only available
to the containers within an Acorn. Expose(d) ports are available to services within the cluster. And
publish ports are available publicly outside the cluster. The access mode defined in the Acornfile is
just the default behavior and can be changed at deploy time.

```acorn
containers: web: {
 image: "nginx"
 
 // Define internal TCP port 80 available internally as DNS web:80
 ports: 80
 
 // Define internal HTTP port 80 available internally as DNS web:80
 // Valid protocols are tcp, udp, and http
 ports: "80/http"
 
 // Define internal HTTP port 80 that maps to the container port 8080
 // available internally as DNS web:80
 ports: "80:8080/http"
 
 // Define internal TCP port 80 that maps to the container port 8080
 // available internally as DNS web:80
 ports: "80:8080"
 
 // Define internal TCP port 80 that maps to the container port 8080
 // available internally as DNS web-alias:80
 ports: "web-alias:80:8080"
 
 // The similar ports as above but just in a list
 ports: [
  80,
     "80/http",
     "80:8080/http",
 ]
 
 ports: {
  // Define publically accessible HTTP port 80 that maps to the container port 8080
     // available publically as a DNS assigned at runtime
  publish: ["80:8080/http"]
  
  // Define cluster accessible HTTP port 80 that maps to the container port 8080
     // available publically as a DNS assigned at runtime
  expose: ["80:8080/http"]
  
     // Define internal HTTP port 80 that maps to the container port 8080
     // available internally as DNS web:80
  internal: ["80:8080/http"]
 }
 
    // Define publically accessible HTTP port 80 that maps to the container port 8080
    // available publically as a DNS assigned at runtime
 ports: publish: "80:8080/http"
}
```

### probes, probe

`probes` configure probes that can signal when the container is ready, alive, and started. There are
three probe types: `readiness`, `liveness`, and `startup`. `readiness` probes indicate when an application
is available to handle requests. Ports will not be accessible until the `readiness` probe passes. `liveness`
probes indicate if a running process is healthy. If the `liveness` probe fails, the container will be deleted
and restarted. `startup` probe indicates that the container has started for the first time.

```acorn
containers: web: {
 image: "nginx"
 
 // Configure readiness probe to run probe.sh
 probe: "probe.sh"
 
 // Configure readiness probe to look for a HTTP 200 response from localhost port 80
 probe: "http://localhost:80"
 
 // Configure readiness probe to connect to TCP port 1234
 probe: "tcp://localhost:1234"
 
 probes: {
  "readiness": {
            // Configure a HTTP readiness probe
            http: {
                url: "http://localhost:80"
                headers: {
                    "X-SOMETHING": "some-value"
                }
            }
            // Below are the default values for the following parameters
            initialDelaySeconds: 0
            timeoutSeconds: 1
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 3
  }
  "liveness": {
            // Configure an Exec liveness probe
            exec: {
                command: ["probe.sh"]
            }
  }
  "startup": {
            // Configure a TCP startup liveness probe
            tcp: {
             url: "tcp://localhost:1234"
            }
  }
 }
}

```

### scale

`scale` configures the number of container replicas based on this configuration that should
be ran.

```acorn
containers: web: {
 image: "nginx"
 scale: 2
}
```

### sidecars

`sidecars` are containers that run colocated with the parent container and share the same network
address. Sidecars accept all the same parameters as a container and one additional parameter `init`

```acorn
containers: web: {
 image: "nginx"
 sidecars: sidecar: {
  image: "someother-image"
 }
}
```

#### init

`init` tells the container runtime that this `sidecar` must be ran first on startup and the main
container will not run until this container is done

```acorn
containers: web: {
    image: "nginx"
    dirs: "/run/startup/": "ephemeral://startup-info"
    sidecars: "stage-data": {
     // This sidecar will run first and only when it exits with exit code 0 will the 
     // parent "web" container start
     init: true
     image: "ubuntu"
        dirs: "/run/startup/": "ephemeral://startup-info"
        command: "./stage-data-to /run/startup"
    }
}
```

### permissions

`permissions` allow you to define what resources the container can interact with on-cluster and what it can do with them.

```acorn
containers: web: {
    image: "nginx"
    permissions: {
  // These are permissions that will only be granted for this container in its namespace.
  rules: [{
   // Configure what actions you can do on the defined resources
   verbs: [
    "get", 
    "list", 
    "watch",
    "create", 
   ]
   // Define what API group the resources belong to
   apiGroups: [
    "api.sample.io"
   ]
   // Configure which resources in the above apiGroups to apply the above verbs to
   resources: [
    "fooresource"
   ]
  }]
  // These are permissions that will be granted for this container in all namespaces.
  clusterRules: [{
   verbs: [
    "get", 
    "list", 
    "watch",
   ]
   apiGroups: [
    "api.sample.io"
   ]
   resources: [
    "fooresource"
   ]
   // Optionally restrict permissions to a specific namespace and not globally
   namespaces: [
    "other-namespace"
   ]
  }]
    }
}
```

### memory

`memory` allows you to specify how much memory the container should run with. It can be abbreviated to `mem`. If left unspecified, it will be defaulted to the compute class default. See [Acorn sizing](production/acorn-sizing) for more info. This should only be set in the Acornfile if there is a minimum memory requirement for the container to run.

```acorn
containers: {
    nginx: {
        image: "nginx"
        ports: publish: "80/http"
        files: {
            "/usr/share/nginx/html/index.html": "<h1>My first Acorn!</h1>"
        }
        memory: 512Mi
    }
}
```

### class

`class` allows you to specify what compute class the container should run on. If left unspecified, it will be defaulted to the project-level default. If there is no project-level default it will use the cluster-level default. If there is no cluster-level default then no compute class will be used. See [Acorn sizing](production/acorn-sizing) for more information. This should only be set if it is required for the container to run, otherwise, portability is reduced.

```acorn
containers: {
    nginx: {
  class: "sample-compute-class"
        image: "nginx"
        ports: publish: "80/http"
        files: {
            "/usr/share/nginx/html/index.html": "<h1>My first Acorn!</h1>"
        }
        memory: 512Mi
    }
}
```

### metrics

`metrics` allows you to specify the HTTP port and path on which the container will expose metrics. Acorn will take this information and create the standard `prometheus.io` scrape annotations on the Kubernetes Pod(s) created for the container.

```acorn
containers: "mycontainer": {
    image: "nginx"
    ports: ["80/http", "8080/http"]
    metrics: {
        port: 8080
        path: "/metrics"
    }
}
```

The Pod created for this container will have the following annotations:

```yaml
prometheus.io/scrape: "true"
prometheus.io/port: "8080"
prometheus.io/path: "/metrics"
```

## services (consuming)

`services` are Acorns that will deploy cloud services outside the scope of Acorn and provide endpoints, credentials, and other information needed for other Acorns to consume the service. These services are typically managed by the cloud provider.  For example, a service could be a RDS database or a S3 bucket.

```acorn
services: rds: {
 image: "ghcr.io/acorn-io/aws/rds-aurora-cluster"
}

containers: web: {
 image: "nginx"
 env: {
 DB_SERVER: "@{service.rds.address}"
 DB_USER: "@{service.rds.secrets.admin.username}"
 DB_PASS: "@{service.rds.secrets.admin.password}"
 } 
}
```

### image

`image` refers to the Acorn image to run to provision and provide the service.

### secrets

`secrets` here should be used to define links from the Acorn apps secrets into the service secrets. For example, if the credentials for a service provider need to be bound into the service Acorn for it to provision a new database.

### autoUpgrade

`autoUpgrade` will automatically upgrade the service Acorn when a new version of the service is available in the registry.

### autoUpgradeInterval

`autoUpgradeInterval` will configure how often to check for a new version of the service Acorn in the registry.

### notifyUpgrade

`notifyUpgrade` is a boolean value that will prompt the user to upgrade the service Acorn when a new version is available in the registry.

### memory, mem

`memory` allows you to define a memory resource limit for the pods running/provisioning the service.

### environment, env

`environment` allows you to define environment variables that will be available to the service Acorn.

### serviceArgs

`serviceArgs` allows you to define arguments that will be passed to the service Acorn.

## services (generating)

`services` when defined in an Acornfile will generate a service that can be consumed by other Acorns follow this schema.

```acorn
services: "my-service": {
    generated: job: "create-service"
}
```

### default

`default` is a boolean value that can be used to specify if this service should be the default service when creating multiple services in the same Acornfile.

### generated

`generated` has a single key `job` that defines which job in the Acornfile will be ran to generate the service. The contents of `/run/secrets/output` will be parsed as an Acornfile and the service will be created from that.

### consumer

Consumer allows you to specify a set of permissions that the consuming services will get. These permissions must be a superset of the final permissions that are created at render time.

For example:

```acorn
services: s3: {
    generated: job: "apply"
    consumer: {
        rules: [{
            verbs: [
                "s3:*",
            ]
            apiGroups: [
                "aws.acorn.io"
            ]
            resources: [
                "*"
            ]
        }]
    }
}
```

When the apply job renders the service, they can create more specific permissions, that are scoped down to the specific bucket that is created.

## services (generated)

The generated service is the definition consumed by other Acorn applications.

```acorn
services: "my-service": {
    address: "https://my-service-url"
    secrets: ["service-credentials"]
    data: {
        "key1": "value1"
        "key2": "value2"
    }
}
// ...
```

### address

`address` is a string that provides the address of the service. This is typically a URL or hostname. **address** and **container** are mutually exclusive.

### Container

`container` is a string that provides the name of the container that is providing the service. This will automatically set the address of the service. **container** and **address** are mutually exclusive.

### ports

`ports` can specifiy a single port or be defined as a list of ports the service endpoint is listening on.

### secrets

`secrets` is a list of secrets that are exposed as part of the service that can be consumed by other Acorns. These secrets can provide API keys or other credentials needed to access the service.

### data

`data` is an object where the service author can provide key/value data needed to consume the service like a database name or other configuration data.

## acorns

`acorns` reference Acorns that have been published to a registry or will be built from a self contained Acornfile outside of the current Acornfile.

```acorn
acorns: {
    "my-acorn": {
        image: "my-acorn"
        publishMode: "defined"
        secrets: ["redis-password:redis-password"]
        autoUpgrade: true
        profiles: "prod"
    }
}
```

### image

`image` is the name of the Acorn image to run. **image** and **build** are mutually exclusive.

### build

`build` provides the information to build the Acorn image from source. **build** and **image** are mutually exclusive.

```acorn
acorns: {
    "my-acorn": {
        build: {
            context: "./acorn-other"
            acornfile: "./acorn-other/Acornfile"
            args: {
                "ARG1": "value1"
            }
        }
        ...
    }
}
```

Where `context` is the directory to build the Acorn image from, `acornfile` is the path to the Acornfile to build from, and `args` is a map of arguments to pass during the build.

### publish

`publish` is a list of one or more ports to publish from the acorn image.

### publishMode

`publishMode` defines the behavior of the defined ports in the Acorn image. The values are `all`, `defined`, and `none`. `all` will publish all ports defined in the Acornfile. `defined` will only publish ports defined as `publish` in the Acornfile. `none` will not publish any ports. The default value is `none`.

### volumes

`volumes` are a list of volume bindings to mount into the Acorn image.

### secrets

`secrets` are a list of secrets to mount into the Acorn image.

### links

`links` are a list of links to other Acorns that will be available to the Acorn image.

### autoUpgrade

`autoUpgrade` is a boolean value that will automatically upgrade the Acorn image when a new version is available in the registry.

### autoUpgradeInterval

`autoUpgradeInterval` is a string that defines how often to check for a new version of the Acorn image in the registry.

### notifyUpgrade

`notifyUpgrade` is a boolean value that will prompt the user to upgrade the Acorn image when a new version is available in the registry.

### deployArgs

`deployArgs` is a map of arguments to pass to the Acorn image when it is deployed.

### profiles

`profiles` is a string that specifies which profile to use when deploying the Acorn image.

## jobs

`jobs` are containers that are run once to completion. If the configuration of the job changes, the will
be ran once again.  All fields that apply to [containers](#containers) also apply to
jobs.

```acorn
jobs: "setup-volume": {
 image: "my-app"
 command: "init-data.sh"
 dirs: "/mnt/data": "data"
}
```

### schedule

`schedule` field will configure your job to run on a cron schedule. The format is the standard cron format.

```
 ┌───────────── minute (0 - 59)
 │ ┌───────────── hour (0 - 23)
 │ │ ┌───────────── day of the month (1 - 31)
 │ │ │ ┌───────────── month (1 - 12)
 │ │ │ │ ┌───────────── day of the week (0 - 6) (Sunday to Saturday;
 │ │ │ │ │                                   7 is also Sunday on some systems)
 │ │ │ │ │                                   OR sun, mon, tue, wed, thu, fri, sat
 │ │ │ │ │
 * * * * *
```

The following shorthand syntaxes are supported

| Entry                   |  Description                                             | Equivalent to |
|-------------------------|------------------------------------------------------------|---------------|
| @yearly (or @annually)  | Run once a year at midnight of 1 January                 | 0 0 1 1 *     |
| @monthly                | Run once a month at midnight of the first day of the month | 0 0 1 **     |
| @weekly                | Run once a week at midnight on Sunday morning             | 0 0 ** 0     |
| @daily (or @midnight)   | Run once a day at midnight                                 | 0 0 ** *     |
| @hourly                | Run once an hour at the beginning of the hour             | 0 ****     |

## routers

`routers` support path based HTTP routing so one can expose multiple containers through a
single published service.  For example, if you have two containers named `auth` and `api`
they would have to be exposed as two different HTTP services like `auth.example.com` and `api.example.com`.
The router feature allows you to expose these two containers as `myapp.example.com/auth` and
`myapp.example.com/api`.

```acorn
routers: myapp: {
    routes: {
        // Short syntax to match all prefixes /auth and route to the auth:8080 container
        "/auth": "auth:8080"

        // Verbose syntax
        "/api": {
            // can be "exact" or "prefix"
            pathType: "exact"
            targetServiceName: "api"
            targetPort: 8081
        }
    }
}

containers: auth: {
    image: "auth"
    ports: "8080/http"
}

containers: api: {
    image: "api"
    ports: "8081/http"
}
```

### pathType

`pathType` describes the matching behavior of the route. Currently, "prefix" and "exact"
are supported. "exact" will require that the path matches the string and "prefix" requires
the path to start with the path string. The short syntax of routes will default to "prefix".

```acorn
routers: myapp: routes: {
    "/api": {
        pathType: "exact"
        targetServiceName: "api"
        targetPort: 8081
    }

    // Default to "prefix" match for short syntax
    "/auth": "auth:8080"
}
```

### targetServiceName

`targetServiceName` is the destination service of the route.  This name typically corresponds to
to a `container` name.  Technically it can be any acorn resource that exposes a service name which
is currently `containers`, `jobs`, `routers` and linked in services.

### targetPort

`targetPort` is the destination port of the route. The target `container` or `job` must have this
port defined or else the traffic will be dropped.  If you are targeting another router, routers
implicitly have the internal port `80`

## volumes

`volumes` store persistent data that can be mounted by containers

```acorn
containers: db: {
 image: "mariadb"
 dirs: "/var/lib/mysql": "data"
}
volumes: data: {
 size: "100G"
 accessModes: "readWriteOnce"
 class: "default"
}
```

### size

`size` configures the default size of the volume to be created.  At deploy-time this value can be
overwritten.

```acorn
volumes: data: {
 // All numbers are assumed to be gigabytes
 size: 100

 // The following suffixes are understood
    // 2^x  - Ki | Mi | Gi | Ti | Pi | Ei
    // 10^x - m | k | M | G | T | P | E
 size: "10G"
}
```

### class

`class` refers to the `storageclass` within kubernetes.

```acorn
volumes: data: {
        // either "default" or a storageclass from `kubectl get sc`
 class: "longhorn"
}
```

### accessModes

`accessModes` configures how a volume can be shared among containers.

```acorn
volumes: data: {
 accessModes: [
  // Only usable by containers on the same node
  "readWriteOnce",
  // Usable by containers across many nodes
  "readWriteMany",
  // Usable by containers across many nodes but read only
  "readOnlyMany",
 ]
}
```

## secrets

`secrets` store sensitive data that should be encrypted as rest.

```acorn
secrets: "my-secret": {
    type: "opaque"
    data: {
        key1: ""
        key2: ""
    }
}
```

### type

The common pattern in Acorn is for secrets to be generated if not supplied. `type`
specifies how the secret can be generated. Refer to [the secrets documentation](/authoring/secrets) for
descriptions of the different secret types and how they are used.

```acorn
secrets: "a-token": {
 // Valid types are "opaque", "token", "basic", "generated", and "template"
 type: "opaque"
}
```

### params

`params` are used to configure the behavior of the secrets generation for different types.
Refer to [the secrets documentation](/authoring/secrets) for
descriptions of the different secret types and how their parameters.

```acorn
secrets: "my-token": {
    type: "token"
    params: {
        length: 32
        characters: "abcdedfhifj01234567890"
    }
}
```

### data

`data` defines the keys and non-sensitive values that will be used by the secret.
Refer to [the secrets documentation](/authoring/secrets) for
descriptions of the different secret types and how to use data keys and values.

```acorn
secrets: {
    "my-template": {
        type: "template"
        data: {
            template: """
            a ${secret://my-secret-data/key} value
            """
        }
    }
    "my-secret-data": {
        type: "opaque"
        data: {
            key: "value"
        }
    }
}
```

## args

`args` defines arguments that can be modified at build or runtime by the user.
Arguments to an Acorn can be standard strings, ints, bools, and other complex types. To define an argument,
specify a name and a default value. The type will be inferred from the default value.

```acorn
args: {
    // A description that will be show to the user in `acorn run image --help`
    myIntVar: 1
    // Another description that will be show to the user in `acorn run image --help`
    myStringVar: "somestring"
    // Another description that will be show to the user in `acorn run image --help`
    myBoolVar: true
}
```

## localData

`localData` is used by the Acornfile author to store values to assist in scripting in the Acornfile. These values are
not directly interpreted by Acorn and are only for the authors use.

```acorn
containers:{
    frontend: {
        // ...
        env: {
            "MY_IMPORTANT_SETTING": localData.myApp.frontendConfig.key
        }
        // ...
    }
    database: {
        // ...
        env: {
            "MY_DATABASE_NAME": localData.myApp.databaseConfig.name
        }
        // ...
    }
}
localData: {
    myApp:{
        frontendConfig: {
            key: "value"
        }
        databaseConfig: {
            name: "db-prod"
        }
    }
}
```

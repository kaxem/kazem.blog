---
title: How to Not kill yourself after deploying Cassandra?
date: 2023-11-26
tags: DevOps
---
There was an issue when I mounted a configmap to the /etc/cassandra/cassandra.yaml pod. I encountered the error: `/etc/cassandra read-only filesystem`.
After attempting various solutions, such as running the image as the cassandra user with uid=999 & gid=999 (specified in the [official cassandra image](https://github.com/docker-library/cassandra/blob/master/5.0/Dockerfile)), using a bash script in the statefulset `command: chmod 755 /etc/cassandra`, or using `command:["-Dcassandra.config=file:///path/to/mounted/cassandra.yaml"]`, the problem persisted. so I tried some ways that changed my life.


I tried 3 different attemptsto deploy it, choose one based on your needs:
1. Deploy Cassandra with operator<br/>
This method is suggested because you're not engaging with cassandra's complexility.
2. Deploy Cassandra with Bitnami Helm<br/>
It's not working for kubernetes <v1.19
3. Deploy cassandra by shooting yourself in the foot
![pepe-cry](/images/Cassandra/pepe-the-frog-sad.gif)

# Deploy Cassandra with operator
1. first of all we need to install cert-manager to deploy cassandra operator
   1. add cert-manager repo: `helm repo add cert-manager https://charts.jetstack.io` 
   2. create a values.yaml to change default values of k8ssandra repo and
      paste following configs in values.yaml:
```
     image:
      repository: <private-registery.io>/jetstack/cert-manager-controller
    webhook:
     image:
      repository: <private-registery.io>/jetstack/cert-manager-webhook
    cainjector:
     image:
      repository: <private-registery.io>/jetstack/cert-manager-cainjector
    acmesolver:
     image:
      repository: <private-registery.io>/jetstack/cert-manager-acmesolver
    startupapicheck:
     image:
      repository: <private-registery.io>/jetstack/cert-manager-ctl 
    
```
   
**why we change repositiry to our registery?**
- <private-registery.io> act like a proxy we don't need to change dns server in case of 403 error.
- <private-registery.io> saves image so for next deploying, pulling is so much faster 
  3. install cert-manager `helm install my-cert-manager2 cert-manager/cert-manager --version 1.0.0 --namespace cert-manager --create-namespace -f values.yaml`            
## install cassandra operator:
To install with cassandra operator and cluster first you should go to `https://artifacthub.io/packages/helm/k8ssandra/cass-operator`
1. add repo with `helm repo add k8ssandra https://helm.k8ssandra.io/`
2. update repo `helm repo update`
3. create a values.yaml to change default values of k8ssandra repo:
paste following configs in values.yaml:
```
    image:
      repository: <private-registery.io>/k8ssandra/cass-operator
      registryOverride: <private-registery.io>

```
4. install cassandra operator: `helm install cassandra-operator k8ssandra/cass-operator --version 0.29.2 --namespace cass-operator --create-namespace -f cass-values.yaml`

## install cassandra cluster
Install k8s cluster helm by using below configs:
```
nameOverride: "dc1"
fullnameOverride: "dc1"
spec:
    clusterName: development
    serverType: cassandra
    serverVersion: "4.1.2"
    managementApiAuth:
    insecure: {}
    size: 1
    storageConfig:
        cassandraDataVolumeClaimSpec:
        storageClassName: nfs-csi
        accessModes:
            - ReadWriteOnce
        resources:
            requests:
            storage: 10Gi
    resources:
    requests:
        memory: 4Gi
        cpu: 1000m
    podTemplateSpec:
    securityContext:
        capabilities:
        add:
        - IPC_LOCK
    spec:
        containers:
        - name: cassandra
    racks:
    - name: rack1
```
After installing I've got this error:
>Error: INSTALLATION FAILED: unable to build kubernetes objects from release manifest: error validating "": error validating data: ValidationError(StatefulSet.spec.template.spec.containers[0].securityContext): unknown field "seccompProfile" in io.k8s.api.core.v1.SecurityContext
>
with a search i found out `seccompProfile` field is not supported in k8s v 1.17 
so i went another way:

# Deploy Cassandra with Bitnami Helm 
Install helm chart with `helm install cass-cluster exaRepo/k8ssandra-cluster --namespace cass-operator -f cass-cluster.yaml`
For install: 
first add repo: `helm repo add bitnami https://charts.bitnami.com/bitnami`
the update helm: `helm repo update `
then create a values file: 

```
image:
  registry: <private-registery.io>
  repository: cassandra
  tag: 4.1.3
resources:
    limits:
        cpu: 2
        memory: 3Gi
    requests:
        cpu: 2
        memory: 2Gi

    maxHeapSize: 2G
    newHeapSize: 1G
```

The limits line is because of an error I've got for pod is not creating so based on this [issue](https://github.com/bitnami/charts/issues/5295) I changed this values.
This method is not working in k8s version <1.19 let's try another way.

---------
# Deploy cassandra by shooting yourself in the foot
**Let's build our custome cassandra image:**
After days of trying I finally give up. no more charts and operators!
I tried to solve main problem "**Mounting Cassandra.yaml into /etc/cassandra/ facing `read-only file system` error**"
I started to write a Docker file that pulls cassandra then start it with `CMD="-Dcassandra.config=/path/to/cassandra.yaml"` 
lets write it:
```
FROM <private-registery.io>/cassandra:5.0
WORKDIR /app
COPY cassandra.yaml .
COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh
CMD ["-Dcassandra.config=file:///app/cassandra.yaml"]
ENTRYPOINT ["docker-entrypoint.sh"]
```

**Extra work:**
This is GitLab ci to build and push image to private registery then use it in statefulset:
```
    cassandra-build-on-merge:
    stage: build
    image: docker.io/docker:stable
    rules:
        - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
        changes:
            - "path/to/image/*"
    script:
        - cd path/to/image/
        - docker image build . -f Dockerfile
        -t registery/cassandra:${CI_COMMIT_SHA}
        -t registery/cassandra:latest
        - docker image push registery/cassandra:${CI_COMMIT_SHA}
        - docker image push registery/cassandra:latest

    staging-cassandra-test:
    stage: deploy
    image:
        name: lachlanevenson/k8s-kubectl:latest
        entrypoint: ["/bin/sh", "-c"]
    only:
        refs:
        - master
        variables:
        - $DEPLOY_TO_STAGING =~ /true/
        changes:
        - "path/to/statefulset/**/*"
    script:
        - cd technologies/Staging
        - sed -ie "s/IMAGE-VERSION/$CI_COMMIT_SHA/g" cassandra-test/cassandra-sts.yaml #### we change image version in sts to CI_COMMIT_SHA
        - kubectl apply -f ./cassandra-test --namespace=twitter
```

after building this image I used it in statefulset.yaml:
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: cassandra-test
  labels:
    app: cassandra-test
spec:
  serviceName: cassandra-test
  replicas: 1
  selector:
    matchLabels:
      app: cassandra-test
  template:
    metadata:
      labels:
        app: cassandra-test
    spec:
      terminationGracePeriodSeconds: 1800
      containers:
      - name: cassandra
        image: registery/cassandra:IMAGE-VERSION
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 7000
          name: intra-node
        - containerPort: 7001
          name: tls-intra-node
        - containerPort: 7199
          name: jmx
        - containerPort: 9042
          name: cql
        resources:
          limits:
            cpu: "500m"
            memory: 6Gi
          requests:
            cpu: "500m"
            memory: 3Gi
        env:
          - name: MAX_HEAP_SIZE
            value: 512M
          - name: HEAP_NEWSIZE
            value: 100M
          - name: CASSANDRA_CLUSTER_NAME
            value: "K8Demo"
          - name: CASSANDRA_DC
            value: "DC1-K8Demo"
          - name: CASSANDRA_RACK
            value: "Rack1-K8Demo"
          - name: POD_IP
            valueFrom:
              fieldRef:
                fieldPath: status.podIP
        securityContext:                                                                    
          allowPrivilegeEscalation: false                                                   
          capabilities:                                                                     
            drop:                                                                           
            - ALL                                                                           
          privileged: false                                                                 
          readOnlyRootFilesystem: false                                                     
          runAsNonRoot: true                                                                
          runAsUser: 999                                                                   

        volumeMounts:
        - name: cassandra-data-test
          mountPath: /var/lib/cassandra

      volumes:
      - name: cassandra-data-test
        persistentVolumeClaim:
          claimName: cassandra-test-pv-claim

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: cassandra-test-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi  # how much is claimed
  storageClassName: standard

```
We should pass pod ip to image with `POD_IP = valueFrom.fieldRef.fieldPath.status.podIP` in statefulset. It's important because we should set `listen_address` in cassandra.yaml to this ip.
[docker-entrypoint.sh](https://github.com/docker-library/cassandra/blob/master/5.0/docker-entrypoint.sh) will change these addresses by default but we need to change this entrypoint for new image we're building.
```
 #!/bin/bash
set -e

# first arg is `-f` or `--some-option`
# or there are no args
if [ "$#" -eq 0 ] || [ "${1#-}" != "$1" ]; then
	set -- cassandra -f "$@"
fi

# allow the container to be started with `--user`
if [ "$1" = 'cassandra' -a "$(id -u)" = '0' ]; then
	find "/etc/cassandra" /var/lib/cassandra /var/log/cassandra \
		\! -user cassandra -exec chown cassandra '{}' +
	exec gosu cassandra "$BASH_SOURCE" "$@"
fi

_ip_address() {
	# scrape the first non-localhost IP address of the container
	# in Swarm Mode, we often get two IPs -- the container IP, and the (shared) VIP, and the container IP should always be first
	ip address | awk '
		$1 != "inet" { next } # only lines with ip addresses
		$NF == "lo" { next } # skip loopback devices
		$2 ~ /^127[.]/ { next } # skip loopback addresses
		$2 ~ /^169[.]254[.]/ { next } # skip link-local addresses
		{
			gsub(/\/.+$/, "", $2)
			print $2
			exit
		}
	'
}

# "sed -i", but without "mv" (which doesn't work on a bind-mounted file, for example)
_sed-in-place() {
	local filename="$1"; shift
	local tempFile
	tempFile="$(mktemp)"
	sed "$@" "$filename" > "$tempFile"
	cat "$tempFile" > "$filename"
	rm "$tempFile"
}

if [ "$1" = 'cassandra' ]; then
	: ${CASSANDRA_RPC_ADDRESS='0.0.0.0'}

	: ${CASSANDRA_LISTEN_ADDRESS='auto'}
	if [ "$CASSANDRA_LISTEN_ADDRESS" = 'auto' ]; then
		CASSANDRA_LISTEN_ADDRESS="$(_ip_address)"
	fi

	: ${CASSANDRA_BROADCAST_ADDRESS="$CASSANDRA_LISTEN_ADDRESS"}

	if [ "$CASSANDRA_BROADCAST_ADDRESS" = 'auto' ]; then
		CASSANDRA_BROADCAST_ADDRESS="$(_ip_address)"
	fi
	: ${CASSANDRA_BROADCAST_RPC_ADDRESS:=$CASSANDRA_BROADCAST_ADDRESS}

	if [ -n "${CASSANDRA_NAME:+1}" ]; then
		: ${CASSANDRA_SEEDS:="cassandra"}
	fi
	: ${CASSANDRA_SEEDS:="$CASSANDRA_BROADCAST_ADDRESS"}

	_sed-in-place "cassandra.yaml" \                      ## default is $CASSANDRA_CONF/cassandra.yaml
		-r 's/(- seeds:).*/\1 "'"$CASSANDRA_SEEDS"'"/'

	for yaml in \
		broadcast_address \
		broadcast_rpc_address \
		cluster_name \
		endpoint_snitch \
		listen_address \
		num_tokens \
		rpc_address \
		start_rpc \
	; do
		var="CASSANDRA_${yaml^^}"
		val="${!var}"
		if [ "$val" ]; then
			_sed-in-place "/etc/cassandra/cassandra.yaml" \
				-r 's/^(# )?('"$yaml"':).*/\2 '"$val"'/'
		fi
	done

	for rackdc in dc rack; do
		var="CASSANDRA_${rackdc^^}"
		val="${!var}"
		if [ "$val" ]; then
			_sed-in-place "/etc/cassandra/cassandra-rackdc.properties" \
				-r 's/^('"$rackdc"'=).*/\1 '"$val"'/'
		fi
	done
fi

exec "$@"
```
I changed `_sed-in-place "$CASSANDRA_CONF/cassandra.yaml"  -r 's/(- seeds:).*/\1 "'"$CASSANDRA_SEEDS"'"/'` to
`_sed-in-place "cassandra.yaml" -r 's/(- seeds:).*/\1 "'"$CASSANDRA_SEEDS"'"/'` This is our custome configuration in /app workdir. 
After building image and deploying cassandra I checked pod to see if my has configs in cassandra.yaml has been added.
Exec into pod then use `cqlsh` to connect to cassandra thenuse  query `SELECT * FROM system_views.settings ;` to see all configs and BOOM there are my configs:
![boom](/images/Cassandra/BOOM.png)

GoodLuck BoyKz!
![pepe-goose](/images/Cassandra/pepe-the-frog-goose-ride.gif)

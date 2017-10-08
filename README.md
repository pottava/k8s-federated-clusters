Kubernetes: cluster federation
---

### 1. Set some variables

```sh
$ export DNS_ZONE=your-domain.com
$ export PROJECT_ID=
$ export COMPUTE_REGION=asia-northeast1
$ export COMPUTE_ZONE=asia-northeast1-a
$ export REGIONS="asia-northeast1 us-east4"
```

### 2. CLI configurations

```sh
$ gcloud config set project $PROJECT_ID
$ gcloud config set compute/region ${COMPUTE_REGION}
$ gcloud config set compute/zone ${COMPUTE_ZONE}
$ gcloud config configurations list
```

### 3. Register DNS records

```sh
$ export PROJECT_KEY=$( echo ${PROJECT_ID} | sed -e 's/\..*//g' \
    | sed -e 's/[^a-zA-Z]//g' ) && echo ${PROJECT_KEY}
$ gcloud dns managed-zones create ${PROJECT_KEY} \
    --description "Kubernetes Federation Zone" \
    --dns-name "${DNS_ZONE}."
```

Associate following name servers with DNS provider's.

```sh
$ gcloud dns record-sets list --zone ${PROJECT_KEY} \
    --name ${DNS_ZONE} --type NS --format="json" \
    | jq -r ".[0].rrdatas[]"
```

### 4. Create clusters

Workaround for RBAC errors.

```
$ gcloud config set container/use_client_certificate True
$ export CLOUDSDK_CONTAINER_USE_CLIENT_CERTIFICATE=True
```

Decide kubernetes' version.

```sh
$ gcloud container get-server-config
$ export K8S_VERSION=
```

Create clusters on each regions.

```sh
$ for region in $( echo ${REGIONS} ); do
    cluster_name=${PROJECT_KEY}-${region}
    gcloud container clusters create "${cluster_name}" --cluster-version "${K8S_VERSION}" \
        --zone "${region}-a" --additional-zones "${region}-b" \
        --machine-type "g1-small" --image-type "COS" --disk-size 10 \
        --num-nodes 1 --enable-autoscaling --min-nodes=1 --max-nodes=2 \
        --enable-cloud-logging --enable-cloud-monitoring --enable-cloud-endpoints \
        --enable-autoupgrade --enable-autorepair \
        --scopes "cloud-platform,storage-ro,logging-write,monitoring-write,service-control,service-management,https://www.googleapis.com/auth/pubsub,https://www.googleapis.com/auth/sqlservice.admin" \
        --labels "fed=worker" \
        --async
  done
$ gcloud beta container clusters create ${PROJECT_KEY}-fed --cluster-version "${K8S_VERSION}" \
      --zone "${COMPUTE_ZONE}" \
      --machine-type "g1-small" --image-type "COS" --disk-size 10 \
      --num-nodes 1 \
      --no-enable-cloud-logging --no-enable-cloud-monitoring --no-enable-cloud-endpoints \
      --enable-autoupgrade --maintenance-window "12:00" --enable-autorepair \
      --scopes "cloud-platform,storage-ro,service-control,service-management,https://www.googleapis.com/auth/ndev.clouddns.readwrite" \
      --labels "fed=manager" \
      --preemptible
```

Save contexts as local configurations.

```sh
$ for region in $( echo ${REGIONS} ); do
    cluster_name=${PROJECT_KEY}-${region}
    gcloud container clusters get-credentials ${cluster_name} --zone "${region}-a"
    sleep 1
    new_name=$( echo ${region}.${PROJECT_KEY//_/-} )
    kubectl config rename-context gke_${PROJECT_ID}_${region}-a_${PROJECT_KEY}-${region} ${new_name}
  done
$ gcloud container clusters get-credentials ${PROJECT_KEY}-fed --zone "${COMPUTE_ZONE}"
$ host_context=$( echo host.${PROJECT_KEY//_/-} )
$ kubectl config rename-context gke_${PROJECT_ID}_${COMPUTE_ZONE}_${PROJECT_KEY}-fed ${host_context}
```

### 5. Install and Join to kubefed

```
$ open https://kubernetes.io/docs/tasks/federation/set-up-cluster-federation-kubefed/#getting-kubefed
```

Check its version.

```
$ kubefed version

Client Version: version.Info{Major:"1", Minor:"8", GitVersion:"v1.8.0" ..
Server Version: version.Info{Major:"1", Minor:"7+", GitVersion:"v1.7.6-gke.1" ..
```

Create the federation control plane service.

```
$ kubefed init kfed --host-cluster-context ${host_context} \
    --dns-zone-name "${DNS_ZONE}." --dns-provider google-clouddns
$ kubectl --context=kfed create ns default
```

### 6. Join clusters into the federation

```
$ for region in $( echo ${REGIONS} ); do
    cluster=$( echo ${region}.${PROJECT_KEY//_/-} )
    kubefed --context=kfed join ${region}-cluster --cluster-context=${cluster} \
        --host-cluster-context=${host_context}
  done
$ kubectl config use-context kfed
$ kubectl get clusters
```

### 7. Create global IP address for HTTP(S) Load Balancing

```
$ gcloud compute addresses create ${PROJECT_KEY}-ipv4 --ip-version IPV4 --global
$ static_ip=$( gcloud compute addresses list --filter "name~'${PROJECT_KEY}-ipv4'" --format json )
$ static_ip_name=$( echo ${static_ip} | jq -r '.[0].name' )
$ external_ip=$( echo ${static_ip} | jq -r '.[0].address' ) && echo $external_ip
```

### 8. Configure Cloud Endpoints

Modify the spec.yaml

```
$ cd $GOPATH/src/github.com/pottava/http-return-everything
$ sed -i -e "s/^  version: .*/  version: \'$( date +%Y%m%d%H%M )\'/" spec.yaml
$ sed -i -e "s/^host: localhost/host: www.${DNS_ZONE}/" spec.yaml
```

After verifing your domain ownership, deploy a new endpoint.

```
$ open https://cloud.google.com/endpoints/docs/openapi/serving-apis-from-domains
$ gcloud service-management enable servicemanagement.googleapis.com
$ gcloud service-management deploy spec.yaml --quiet
```

Deploy the modified deployment.

```
$ service_name=
$ config_id=$( gcloud service-management configs list \
    --service ${service_name} --format=json \
    | jq "map(select(.name | index(\"${service_name}\")).id)" \
    | jq -r "sort | .[-1]" \
  ) && echo ${config_id}
$ sed -i -e "s/SERVICE_NAME/${service_name}/" resources/deployment.yaml
$ sed -i -e "s/SERVICE_CONFIG_ID/${config_id}/" resources/deployment.yaml
```

### 9. Deploy sample services to the clusters

```
$ kubectl apply -f resources/deployment.yaml
$ kubectl apply -f resources/service.yaml
$ sed -e "s/\<static-ip-address-name\>/${static_ip_name}/" \
    resources/ingress.yaml.template > resources/ingress.yaml
$ kubectl apply -f resources/ingress.yaml
```

Wait about 10 min.

```
$ for region in $( echo ${REGIONS} ); do
    cluster=$( echo ${region}.${PROJECT_KEY//_/-} )
    echo "\n=============== ${cluster} ==============="
    kubectl --context=${cluster} get all,ing -o wide
  done
```

### 10. Check its behavior

```
$ open http://${external_ip}
$ open https://www.geoscreenshot.com/
```

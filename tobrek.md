1. sidecar
cd ingress-nginx/images/opentelemetry/rootfs
docker build . --tag tobrek/tobrek-nginx:opentelemetry-sidecar-0.1.0

2. nginx image
cd ingress-nginx/images/nginx/rootfs
docker build . --tag tobrek/tobrek-nginx:nginx-0.101.42

3. controller 
cd ingress-nginx
make image REGISTRY=tobrek/tobrek-nginx BASE_IMAGE=tobrek/tobrek-nginx:nginx-0.101.42
docker tag xommaterials/xom-nginx/controller:v0.101.42 xommaterials/xom-nginx:v0.101.42


4. otel config as configmap
{
    "kind": "ConfigMap",
    "apiVersion": "v1",
    "metadata": {
        "name": "nginx-otel",
        "namespace": "ingress-nginx",
        "creationTimestamp": null
    },
    "data": {
        "otel-nginx.toml": "exporter = \"otlp\"\nprocessor = \"batch\"\n\n[exporters.otlp]\n# Alternatively the OTEL_EXPORTER_OTLP_ENDPOINT environment variable can also be used.\nhost = \"http://otel-collector:4317\"\nport = 4317\n\n[processors.batch]\nmax_queue_size = 2048\nschedule_delay_millis = 5000\nmax_export_batch_size = 512\n\n[service]\nname = \"nginx-proxy\" # Opentelemetry resource name\n\n[sampler]\nname = \"AlwaysOn\" # Also: AlwaysOff, TraceIdRatioBased\nratio = 0.1\nparent_based = false\n"
    }
}

5. helm values file:
## nginx configuration
## Ref: https://github.com/kubernetes/ingress-nginx/blob/main/docs/user-guide/nginx-configuration/index.md
##


controller:
  # name: controller
  image:
    registry: tobrek
    image: tobrek-nginx
    tag: v0.101.42
    digest: ''

  extraModules: 
  # []
  ## Modules, which are mounted into the core nginx image
  - name: opentelemetry
    image: "tobrek/tobrek-nginx:opentelemetry-sidecar-0.1.0"
  # The image must contain a `/usr/local/bin/init_module.sh` executable, which
  # will be executed as initContainers, to move its config files within the
  # mounted volume.

  service:
    type: NodePort

  hostPort:
  # needed for getting minikube working
    enabled: true

  extraVolumeMounts:
  # mount opentelemetry-cpp lib config from configmap
  - name: otel-nginx
    readOnly: true
    mountPath: /conf

  extraVolumes:
  # needed for otel-nginx extraVolumeMount
   - name: otel-nginx
     configMap:
      name: nginx-otel


6. helm install
helm upgrade nginx-service ingress-nginx/ingress-nginx --version=4.1.1 --install --namespace=ingress-nginx --create-namespace --values=nginx/helm/minikube-nginx-values.yaml
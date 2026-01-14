# Istio Migration Guide (DigitalOcean Kubernetes)

This guide outlines the steps to migrate your application from Nginx Ingress to Istio Service Mesh on a DigitalOcean Kubernetes (DOKS) cluster using Helm.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Step 1: Install Istio using Helm](#step-1-install-istio-using-helm)
- [Step 2: Wait for LoadBalancer](#step-2-wait-for-loadbalancer)
- [Step 3: Configure Namespace for Sidecar Injection](#step-3-configure-namespace-for-sidecar-injection)
- [Step 4: Create Istio Traffic Resources](#step-4-create-istio-traffic-resources)
- [Step 5: Verify & Migrate](#step-5-verify--migrate)
- [Step 6: DNS Cutover](#step-6-dns-cutover)

## Prerequisites

Ensure you have the following CLI tools installed:

- `kubectl`: configured to access your DOKS cluster (`doctl kubernetes cluster kubeconfig save <cluster_name>`)
- `helm`: for installing Istio charts
- `doctl`: DigitalOcean CLI (optional, but good for checking LB status)

## Step 1: Install Istio using Helm

We will install Istio in the `istio-system` namespace. This includes the base CRDs, the control plane (`istiod`), and the ingress gateway.

### 1. Add the Istio Helm repository

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

> **Note:** This guide installs Istio version **1.27.0** (stable). To check available versions, run: `helm search repo istio --versions`

### 2. Create the system namespace

```bash
kubectl create namespace istio-system
```

### 3. Install Istio Base (CRDs)

```bash
helm install istio-base istio/base -n istio-system \
  --version 1.27.0 \
  --set defaultRevision=default
```

### 4. Install Istio Discovery (Control Plane)

```bash
helm install istiod istio/istiod -n istio-system \
  --version 1.27.0 \
  --wait \
  --set meshConfig.accessLogFile=/dev/stdout \
  --set meshConfig.accessLogEncoding=JSON
```

> **Note:** This configuration enables access logging for the entire mesh, including the Ingress Gateway.

### 5. Install Istio Ingress Gateway

This step creates the LoadBalancer on DigitalOcean. We create a separate namespace for the gateway for better isolation, though it can also reside in `istio-system`.

```bash
kubectl create namespace istio-ingress
helm install istio-ingress istio/gateway -n istio-ingress \
  --version 1.27.0 \
  --set service.type=LoadBalancer \
  --set service.externalTrafficPolicy=Local \
  --set service.annotations."service\.beta\.kubernetes\.io/do-loadbalancer-name"="istio-ingress-lb" \
  --set env.ISTIO_META_ROUTER_MODE=sni-dnat \
  --set env.PILOT_SKIP_VALIDATE_TRUST_DOMAIN=true
```

> **Note:** The annotation helps identify the LB in the DigitalOcean dashboard. Access logs are enabled via the global config in Step 4. The `externalTrafficPolicy: Local` preserves source IPs, but we also need to configure Envoy to use X-Forwarded-For headers (see below).

**Verify external traffic policy is set to Local:**

```bash
kubectl describe svc istio-ingress -n istio-ingress | grep -A2 -B2 "External Traffic Policy"
```

### 6. Configure Gateway to Preserve Real Client IP

Create an EnvoyFilter to configure the gateway to trust and use X-Forwarded-For headers from the DigitalOcean LoadBalancer.

The configuration file `gateway-client-ip-config.yaml` is already included in this repository. Apply it:

```bash
kubectl apply -f gateway-client-ip-config.yaml
```

This configuration:
- Sets `xff_num_trusted_hops: 1` to trust the X-Forwarded-For header from the DigitalOcean LoadBalancer
- Enables `use_remote_address: true` to use the real client IP
- Updates the access log format to include `x_forwarded_for` and `downstream_remote_address` fields to see the real client IP

**Restart the gateway pods to apply the changes:**

```bash
kubectl rollout restart deployment istio-ingress -n istio-ingress
```

**After restart, check the logs to verify the real client IP is being logged:**

```bash
kubectl logs -n istio-ingress -l istio=ingressgateway --tail=50 | jq '.downstream_remote_address, .x_forwarded_for'
```

### 7. Install Observability Add-ons (Prometheus & Kiali)

Kiali requires Prometheus to visualize the mesh.

#### Install Prometheus

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.27/samples/addons/prometheus.yaml
```

> **Note:** Alternatively, you can use the stable `prometheus-community/prometheus` helm chart, but the sample yaml is the quickest way to get started compatible with Istio.

#### Install Kiali

```bash
helm repo add kiali https://kiali.org/helm-charts
helm repo update
helm install \
    --namespace istio-system \
    --set auth.strategy="anonymous" \
    --set external_services.prometheus.url="http://prometheus.istio-system:9090" \
    kiali-server \
    kiali/kiali-server
```

#### Install Grafana

You have two options:

**Option 1: Install Grafana with Istio Dashboards (Recommended)**

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.27/samples/addons/grafana.yaml
```

> This installs Grafana with all Istio dashboards pre-configured and connected to Prometheus.

**Option 2: Install Grafana via Helm (then import dashboards manually)**

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install \
    --namespace istio-system \
    --set persistence.enabled=true \
    --set adminPassword=admin \
    --set service.type=ClusterIP \
    grafana \
    grafana/grafana
```

> If using Helm, you'll need to import Istio dashboards manually (see below).

#### Access Kiali Dashboard

```bash
istioctl dashboard kiali
# OR if you don't have istioctl installed:
kubectl port-forward svc/kiali -n istio-system 20001:20001
# Then visit http://localhost:20001
```

#### Access Grafana Dashboard

```bash
istioctl dashboard grafana
# OR if you don't have istioctl installed:
kubectl port-forward svc/grafana -n istio-system 3000:80
# Then visit http://localhost:3000
# Default credentials: admin / admin (change on first login)
```

#### Import Istio Dashboards (if using Helm installation)

If you installed Grafana via Helm, you need to import the official Istio dashboards to monitor all traffic:

1. **Configure Prometheus Data Source:**
   - Go to Configuration > Data Sources
   - Add Prometheus data source
   - URL: `http://prometheus.istio-system:9090`
   - Click "Save & Test"

2. **Import Istio Dashboards:**
   - Go to Dashboards > Import
   - Import the following official Istio dashboards by ID:
     - **Istio Service Dashboard**: ID `7636` - Monitor individual service traffic, request rates, error rates, latency
     - **Istio Workload Dashboard**: ID `7639` - Monitor workload-level metrics and performance
     - **Istio Mesh Dashboard**: ID `7630` - Overview of all services in the mesh
     - **Istio Performance Dashboard**: ID `11829` - Resource usage and performance metrics
     - **Istio Control Plane Dashboard**: ID `7645` - Control plane health and metrics
     - **Istio Wasm Extension Dashboard**: ID `13277` - WebAssembly extension metrics
   
   Alternatively, download JSON files directly:
   ```bash
   # Download dashboard JSON files
   curl -o istio-service-dashboard.json https://grafana.com/api/dashboards/7636/revisions/latest/download
   curl -o istio-workload-dashboard.json https://grafana.com/api/dashboards/7639/revisions/latest/download
   curl -o istio-mesh-dashboard.json https://grafana.com/api/dashboards/7630/revisions/latest/download
   curl -o istio-performance-dashboard.json https://grafana.com/api/dashboards/11829/revisions/latest/download
   curl -o istio-control-plane-dashboard.json https://grafana.com/api/dashboards/7645/revisions/latest/download
   ```
   Then import each JSON file via Grafana UI: Dashboards > Import > Upload JSON file

> **Note:** If you used Option 1 (Istio addon), all dashboards are already pre-configured and available in Dashboards > Browse.

#### Import Custom HTTP Status Codes Dashboard

This custom dashboard shows 2XX, 4XX, and 5XX status codes with namespace filtering:

1. **Import the Dashboard:**
   - Go to Dashboards > Import
   - Click "Upload JSON file"
   - Select the file: `dashboards/istio-http-status-codes-dashboard.json`
   - Or copy the JSON content from the file and paste it in the "Import via panel json" text area
   - Click "Load"

2. **Select Data Source:**
   - Grafana will prompt you to select a Prometheus data source
   - Choose your Prometheus datasource (e.g., the one pointing to `http://prometheus.istio-system:9090`)
   - If you see multiple Prometheus datasources, select the one configured for Istio metrics
   - Click "Import"
   - > **Note:** If you get a "Failed to upgrade legacy queries" error, make sure you have a Prometheus datasource configured first in Configuration > Data Sources

3. **Using the Dashboard:**
   - **Namespace Filter**: Use the dropdown at the top to select specific namespaces or "All" to see all namespaces
   - **Panels**:
     - **HTTP Status Codes by Service (Requests per Second)**: Time series showing 2XX (green), 4XX (yellow), and 5XX (red) rates
     - **Current Request Rates by Service**: Table showing current request rates per service
     - **Total HTTP Status Codes (Count)**: Total count of status codes over the selected time range
     - **HTTP Status Codes Summary Table**: Detailed table with rates for each service, sorted by 5XX errors
     - **Total Request Counts**: Total request counts for 2XX, 4XX, and 5XX
     - **Request Percentage Breakdown**: Percentage of requests that are 2XX, 4XX, and 5XX
     - **Detailed Request Counts by Response Code**: Individual HTTP response codes (200, 404, 500, etc.)
     - **Request Paths by Status Code**: Shows which request paths/endpoints (e.g., `/api/health`, `/api/users`) are returning which status codes

> This dashboard helps you quickly identify services with high error rates (4XX/5XX) across all or selected namespaces.

**Note on Request Paths Panel:** The "Request Paths by Status Code" panel requires the `request_url` label to be available in Istio metrics. If the panel shows no data, you may need to enable request path tracking in your Istio Telemetry configuration. To enable it, create a Telemetry resource:

```yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: request-path-metrics
  namespace: istio-system
spec:
  metrics:
  - providers:
    - name: prometheus
    overrides:
    - match:
        metric: REQUEST_COUNT
      tagOverrides:
        request_url:
          value: "{{.request.url_path}}"
```

Apply it with: `kubectl apply -f request-path-telemetry.yaml`

## Step 2: Wait for LoadBalancer

Wait for the DigitalOcean LoadBalancer to be provisioned and assigned an IP address.

1. **Check the service status:**
   ```bash
   kubectl get svc -n istio-ingress istio-ingress
   ```
   Wait until the `EXTERNAL-IP` field shows an IP address (not `<pending>`).

2. **Save this IP address.** You will need it for DNS updates later.

## Step 3: Configure Namespace for Sidecar Injection

For your application components to be part of the mesh, they need the Envoy sidecar proxy.

1. **Label your application namespace** (replace `default` with your app's namespace if different):
   ```bash
   kubectl label namespace default istio-injection=enabled
   ```

2. **Restart your application pods** to inject the sidecar:
   ```bash
   kubectl rollout restart deployment <your-deployment-name> -n default
   ```
   > Check that pods now have 2 containers (APP + istio-proxy).

## Step 4: Create Istio Traffic Resources

You need to tell Istio how to route traffic to your service. This replaces your Nginx `Ingress` resource.

### 1. Create a Gateway Resource

This configures the LoadBalancer to accept traffic for your hosts.

Create a file named `gateway.yaml`:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: my-app-gateway
  namespace: default
spec:
  selector:
    istio: ingress # matches the label of the istio-ingress helm chart
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*" # You can restrict this to your specific domain
```

Apply it:

```bash
kubectl apply -f gateway.yaml
```

### 2. Create a VirtualService Resource

This routes traffic from the Gateway to your internal Kubernetes Service.

Create a file named `virtual-service.yaml`:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-app-vs
  namespace: default
spec:
  hosts:
  - "*"
  gateways:
  - my-app-gateway
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: <your-service-name> # e.g., my-app-service
        port:
          number: 80 # The port your Service listens on
```

Apply it:

```bash
kubectl apply -f virtual-service.yaml
```

## Step 5: Verify & Migrate

At this point, you have two Ingress points active:

1. **Old**: Nginx Ingress (Likely your current DNS points here)
2. **New**: Istio Ingress (Points to the new DigitalOcean LB IP)

**To Verify:**

Run a curl command against the **new Istio IP** directly:

```bash
curl -v -H "Host: your-domain.com" http://<ISTIO_INGRESS_IP>/
```

You should see your application response. The headers might include `server: istio-envoy`.

## Step 6: DNS Cutover

Once you confirmed the application works via the new IP:

1. Update your DNS A record to point to the **Istio Ingress IP**.
2. Wait for DNS propagation.
3. Once traffic has fully shifted, you can optionally uninstall Nginx Ingress.

---

## Additional Resources

- [Istio Documentation](https://istio.io/latest/docs/)
- [DigitalOcean Kubernetes Documentation](https://docs.digitalocean.com/products/kubernetes/)
- [Istio Helm Charts](https://github.com/istio/istio/tree/master/manifests/charts)

## Troubleshooting

### Gateway not receiving real client IP

If you're still seeing LoadBalancer IPs instead of client IPs in logs:

1. Verify `externalTrafficPolicy: Local` is set:
   ```bash
   kubectl describe svc istio-ingress -n istio-ingress | grep "External Traffic Policy"
   ```

2. Check that the EnvoyFilter is applied:
   ```bash
   kubectl get envoyfilter -n istio-ingress
   ```

3. Verify gateway pods have restarted:
   ```bash
   kubectl get pods -n istio-ingress
   ```

4. Check logs for client IP:
   ```bash
   kubectl logs -n istio-ingress -l istio=ingressgateway --tail=100 | jq '.downstream_remote_address'
   ```

### Dashboard not showing data

- Ensure Prometheus is running and scraping metrics
- Verify the Prometheus data source is correctly configured in Grafana
- Check that services have Istio sidecars injected
- Verify the namespace filter in the dashboard matches your actual namespaces

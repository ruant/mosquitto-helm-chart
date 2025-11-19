# Mosquitto Kubernetes Helm Chart

A comprehensive Kubernetes Helm Chart for Eclipse Mosquitto MQTT Broker with advanced TLS and listener configuration.

## Features

- ✅ **Granular listener control** - Enable/disable MQTT, WebSocket, and TLS listeners independently
- ✅ **Comprehensive TLS support** - Direct TLS termination by Mosquitto with cert-manager integration
- ✅ **Flexible WebSocket options** - Choose between ingress TLS or direct Mosquitto TLS
- ✅ **Service annotations** - Full support for load balancers, monitoring, and service mesh
- ✅ **Advanced security** - Mutual TLS, client certificates, custom cipher suites
- ✅ **Production ready** - Resource limits, security contexts, health checks

## Installation

Install the helm chart repository:

```bash
helm repo add mosquitto https://sintef.github.io/mosquitto-helm-chart
```

Install the chart:

```bash
helm install mosquitto mosquitto/mosquitto -f your-values.yaml
```

## Quick Start Examples

### Basic MQTT (Development)
```bash
helm install mosquitto mosquitto/mosquitto
```

### Secure MQTT with cert-manager
```bash
helm install mosquitto mosquitto/mosquitto \
  --set listeners.mqtt.enabled=false \
  --set listeners.mqttTls.enabled=true \
  --set listeners.mqttTls.tls.secretName=my-mqtt-cert
```

### WebSocket with ingress
```bash
helm install mosquitto mosquitto/mosquitto \
  --set listeners.websocket.ingress.enabled=true \
  --set listeners.websocket.ingress.hosts[0].host=mqtt.example.com
```

## Configuration

### Core Configuration

| Property | Description | Default |
| -------- | ----------- | ------- |
| `image.repository` | Container image repository | `eclipse-mosquitto` |
| `image.tag` | Container image tag | `2.0.22` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `general.maxInflightMessages` | Maximum in-flight messages (0 = unlimited) | `20` |
| `general.maxQueuedMessages` | Maximum queued messages (0 = unlimited) | `1000` |

### Listener Configuration

| Property | Description | Default |
| -------- | ----------- | ------- |
| `listeners.mqtt.enabled` | Enable plain MQTT listener (port 1883) | `true` |
| `listeners.mqtt.port` | Plain MQTT port | `1883` |
| `listeners.websocket.enabled` | Enable WebSocket listener | `true` |
| `listeners.websocket.port` | WebSocket port | `9001` |
| `listeners.mqttTls.enabled` | Enable MQTT over TLS listener | `false` |
| `listeners.mqttTls.port` | MQTT TLS port | `8883` |

### WebSocket TLS Configuration

| Property | Description | Default |
| -------- | ----------- | ------- |
| `listeners.websocket.tls.enabled` | Enable direct WebSocket TLS | `false` |
| `listeners.websocket.tls.secretName` | Kubernetes secret with TLS certificate | `""` |
| `listeners.websocket.tls.certKey` | Certificate key in secret | `tls.crt` |
| `listeners.websocket.tls.keyKey` | Private key in secret | `tls.key` |
| `listeners.websocket.tls.ciphers` | Custom cipher suites (TLS 1.2) | `""` |
| `listeners.websocket.tls.ciphersTls13` | Custom cipher suites (TLS 1.3) | `""` |

### WebSocket Ingress Configuration

| Property | Description | Default |
| -------- | ----------- | ------- |
| `listeners.websocket.ingress.enabled` | Enable ingress for WebSocket | `false` |
| `listeners.websocket.ingress.className` | Ingress class name | `""` |
| `listeners.websocket.ingress.annotations` | Ingress annotations | `{}` |
| `listeners.websocket.ingress.hosts` | Ingress hosts configuration | `[]` |
| `listeners.websocket.ingress.tls` | Ingress TLS configuration | `[]` |

### MQTT TLS Configuration

| Property | Description | Default |
| -------- | ----------- | ------- |
| `listeners.mqttTls.tls.secretName` | Kubernetes secret with TLS certificate | `""` |
| `listeners.mqttTls.tls.certKey` | Certificate key in secret | `tls.crt` |
| `listeners.mqttTls.tls.keyKey` | Private key in secret | `tls.key` |
| `listeners.mqttTls.tls.caKey` | CA certificate key in secret | `ca.crt` |
| `listeners.mqttTls.tls.minVersion` | Minimum TLS version | `tlsv1.2` |
| `listeners.mqttTls.tls.requireClientCerts` | Require client certificates (mTLS) | `false` |
| `listeners.mqttTls.tls.useClientCertAsUsername` | Use client cert CN as username | `false` |
| `listeners.mqttTls.tls.ciphers` | Custom cipher suites (TLS 1.2) | `""` |
| `listeners.mqttTls.tls.ciphersTls13` | Custom cipher suites (TLS 1.3) | `""` |

### Authentication Configuration

| Property | Description | Default |
| -------- | ----------- | ------- |
| `auth.enabled` | Enable authentication (disable for anonymous) | `true` |
| `auth.usersExistingSecret` | Use existing secret for user passwords | `""` |
| `auth.users[].username` | Username | `admin` |
| `auth.users[].password` | Password (Mosquitto sha512-pbkdf2 format) | `admin` |
| `auth.users[].acl[].topic` | ACL topic pattern | `#` |
| `auth.users[].acl[].access` | ACL access level (read/write/readwrite) | `readwrite` |

### Service Configuration

This chart creates **separate services** for different protocols to optimize networking:

#### MQTT Service Configuration
| Property | Description | Default |
| -------- | ----------- | ------- |
| `service.mqtt.type` | MQTT service type (ClusterIP/LoadBalancer/NodePort) | `LoadBalancer` |
| `service.mqtt.annotations` | MQTT service annotations (for load balancer config) | `{}` |

#### WebSocket Service Configuration
| Property | Description | Default |
| -------- | ----------- | ------- |
| `service.websocket.type` | WebSocket service type (ClusterIP/LoadBalancer/NodePort) | `ClusterIP` |
| `service.websocket.annotations` | WebSocket service annotations (for ingress/monitoring) | `{}` |

**Service Architecture:**
- **`mosquitto-mqtt`** service: Handles ports 1883 (MQTT) and 8883 (MQTT TLS) for direct client connections
- **`mosquitto-websocket`** service: Handles port 9001 (WebSocket) for ingress routing or direct access

**Why separate services?**
- MQTT TLS requires direct LoadBalancer access (can't route through HTTP ingress)
- WebSocket can use ClusterIP + ingress for cost-effective routing
- Allows running both protocols simultaneously with different networking approaches

**Common Service Configurations:**

For **MQTT-focused deployments** (IoT sensors, direct MQTT clients):
```yaml
service:
  mqtt:
    type: LoadBalancer          # Direct access to MQTT/TLS ports
    annotations:
      metallb.universe.tf/address-pool: "production-ips"
  websocket:
    type: ClusterIP             # Optional WebSocket for web clients
```

For **WebSocket-focused deployments** (web applications):
```yaml
service:
  mqtt:
    type: ClusterIP             # Internal MQTT access only
  websocket:
    type: LoadBalancer          # Direct WebSocket access
    # OR use ClusterIP + ingress for HTTPS termination
```

### Resource Configuration

| Property | Description | Default |
| -------- | ----------- | ------- |
| `resources.limits.cpu` | CPU limit | `""` |
| `resources.limits.memory` | Memory limit | `""` |
| `resources.requests.cpu` | CPU request | `""` |
| `resources.requests.memory` | Memory request | `""` |
| `nodeSelector` | Node selector | `{}` |
| `tolerations` | Pod tolerations | `[]` |
| `affinity` | Pod affinity | `{}` |

## Passwords

Use the software `mosquitto_passwd` to generate the password in the correct format:

```bash
FILE=$(mktemp)
mosquitto_passwd -H sha512-pbkdf2 -c $FILE USERNAME
cut -d ":" -f 2 $FILE
rm $FILE
```

## Deployment Examples

### 1. Development Setup (Default)
Simple MQTT broker with authentication:

```yaml
# Default configuration - good for development
listeners:
  mqtt:
    enabled: true      # Plain MQTT on port 1883
  websocket:
    enabled: true      # WebSocket on port 9001
  mqttTls:
    enabled: false     # No TLS

auth:
  enabled: true        # Password authentication
```

### 2. Production Setup with cert-manager
Secure deployment with **both** WebSocket ingress **and** MQTT TLS simultaneously:

```yaml
# Secure production deployment - both protocols enabled
listeners:
  mqtt:
    enabled: false           # Disable plain text
  websocket:
    enabled: true
    ingress:
      enabled: true          # WebSocket via ingress with TLS
      className: "nginx"
      annotations:
        cert-manager.io/cluster-issuer: "letsencrypt-prod"
      hosts:
        - host: mqtt-ws.example.com
          paths:
            - path: /
              pathType: Prefix
      tls:
        - secretName: mqtt-websocket-tls
          hosts:
            - mqtt-ws.example.com
  mqttTls:
    enabled: true            # Direct MQTT TLS
    tls:
      secretName: "mqtt-server-tls"
      minVersion: "tlsv1.2"

service:
  # MQTT service for direct TLS connections (8883)
  mqtt:
    type: LoadBalancer
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
      service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"

  # WebSocket service for ingress routing (9001)
  websocket:
    type: ClusterIP           # Ingress handles external access
    annotations:
      prometheus.io/scrape: "true"
      prometheus.io/port: "9001"
```

### 3. Mutual TLS (mTLS) Setup
High security with client certificates:

```yaml
listeners:
  mqtt:
    enabled: false
  mqttTls:
    enabled: true
    tls:
      secretName: "mqtt-mtls-cert"
      requireClientCerts: true          # Mutual TLS
      useClientCertAsUsername: true     # Use cert CN as username
      caKey: "ca.crt"                   # Client CA certificate

auth:
  enabled: false  # Authentication via client certificates
```

### 4. Both WebSocket TLS Options
WebSocket with direct TLS (alternative to ingress):

```yaml
listeners:
  websocket:
    enabled: true
    tls:
      enabled: true                     # Direct WebSocket TLS
      secretName: "websocket-tls-cert"
    ingress:
      enabled: false                    # No ingress needed
```

### 5. Multi-Protocol Setup
All protocols with different certificates:

```yaml
listeners:
  mqtt:
    enabled: true               # Plain MQTT for internal
  websocket:
    enabled: true
    tls:
      enabled: true
      secretName: "ws-cert"     # WebSocket TLS certificate
    ingress:
      enabled: false
  mqttTls:
    enabled: true
    tls:
      secretName: "mqtt-cert"   # Different cert for MQTT TLS
```

## TLS Configuration

This Helm chart provides comprehensive TLS support with multiple configuration options:

### Direct TLS Support
- ✅ **MQTT over TLS** (port 8883) - Full TLS termination by Mosquitto
- ✅ **WebSocket over TLS** - Direct TLS termination by Mosquitto
- ✅ **Mutual TLS** - Client certificate authentication
- ✅ **Custom cipher suites** - Fine-grained TLS control
- ✅ **cert-manager integration** - Automatic certificate management

### WebSocket TLS Options
1. **Ingress TLS** (Recommended): Let nginx/ingress handle TLS termination
2. **Direct TLS**: Mosquitto handles WebSocket TLS directly

### TLS Features
- Support for TLS 1.1, 1.2, and 1.3
- Custom cipher suite configuration
- Client certificate validation
- CA certificate support
- Separate certificates per protocol
- cert-manager secret integration

## cert-manager Integration

This chart works seamlessly with cert-manager for automatic certificate management.

### Example Certificate Resource
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: mqtt-server-tls
spec:
  secretName: mqtt-server-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - mqtt.example.com
```

### Using the Certificate
```yaml
listeners:
  mqttTls:
    enabled: true
    tls:
      secretName: "mqtt-server-tls"  # Matches cert-manager secret
```

## Troubleshooting

### TLS Certificate Issues
- Ensure the secret exists in the same namespace as Mosquitto
- Verify certificate and key are valid: `kubectl describe secret mqtt-server-tls`
- Check certificate expiration: `openssl x509 -in cert.pem -text -noout`

### Connection Issues
- Plain MQTT: `mosquitto_pub -h MQTT_HOST -p 1883 -t test -m "hello"`
- MQTT over TLS: `mosquitto_pub -h MQTT_HOST -p 8883 --cafile ca.crt -t test -m "hello"`
- WebSocket: Use browser developer tools or `wscat`

### Health Check Failures
- Health checks use the first available listener port
- If all listeners are disabled, deployment will fail
- Ensure at least one listener is enabled

## Security Considerations

### Production Recommendations
1. **Disable plain MQTT**: Set `listeners.mqtt.enabled: false`
2. **Enable TLS**: Use `listeners.mqttTls.enabled: true`
3. **Use strong TLS**: Set `minVersion: "tlsv1.2"` or higher
4. **Resource limits**: Set appropriate CPU and memory limits
5. **Network policies**: Restrict network access as needed
6. **Regular updates**: Keep Mosquitto image updated

### Authentication Options
1. **Password authentication**: Default, uses `auth.users[]`
2. **Client certificates**: Set `requireClientCerts: true`
3. **Secret**: Use `auth.usersExistingSecret`
4. **Anonymous access**: Set `auth.enabled: false` (not recommended)

## Contributing

This Helm chart is open source. Contributions are welcome!

### Development
```bash
git clone https://github.com/ruant/mosquitto-helm-chart
cd mosquitto-helm-chart
helm lint charts/mosquitto
helm template mosquitto charts/mosquitto --debug
```

### Testing
```bash
# Test different configurations
helm template mosquitto charts/mosquitto --set listeners.mqtt.enabled=false
helm template mosquitto charts/mosquitto --set listeners.mqttTls.enabled=true
```
# Go extension

This extension provides Go code intelligence on Sourcegraph.

![gif](https://cl.ly/1A0m0L3w2K2g/go.gif)

## Usage on Sourcegraph.com

It's [enabled by default](https://sourcegraph.com/extensions/chris/lang-go)!

## Usage on Sourcegraph 3.x instances

- Enable this extension in the extension registry https://sourcegraph.example.com/extensions
- [Deploy go-langserver](#deploying-the-server)
- Set `go.serverUrl` to the address of your go-langserver instance
- Visit a Go file and you should see code intelligence

This extension communicates with an instance of the [go-langserver](https://github.com/sourcegraph/go-langserver) over WebSockets.

## Usage on Sourcegraph 2.x instances

- Enable this extension in the extension registry https://sourcegraph.example.com/extensions
- Enable the Go language server is running (check https://sourcegraph.example.com/site-admin/code-intelligence)
- Visit a Go file and you should see code intelligence

This extension hits the LSP gateway on the Sourcegraph instance (internally it uses [langserver-http](https://github.com/sourcegraph/sourcegraph-langserver-http)).

## Deploying the server

⚠️ Currently, the language server must be deployed in the same cluster as the Sourcegraph frontend because it has code dependencies on gitserver (for backcompat) which will be eliminated once the buildserver code is moved to go-langserver. See the [tracking issue](https://github.com/sourcegraph/sourcegraph/issues/958).

(untested!) Locally with Docker (expects a sibling container `sourcegraph-frontend-internal`):

```
docker run --rm -p 7777:7777 sourcegraph/xlang-go:23745_2018-11-16_484f19d
```

On Kubernetes (this is running on Sourcegraph.com):

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/port: "6060"
    prometheus.io/scrape: "true"
  labels:
    app: lang-go
  name: lang-go
  namespace: prod
spec:
  loadBalancerIP: your.static.ip.address
  ports:
  - name: debug
    port: 6060
    targetPort: debug
  - name: lsp
    port: 443
    targetPort: lsp
  selector:
    app: lang-go
  type: LoadBalancer
```

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    description: Go code intelligence provided by xlang-go, but supporting TLS and
      WebSockets.
  name: lang-go
  namespace: prod
spec:
  minReadySeconds: 10
  replicas: 1
  revisionHistoryLimit: 10
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: lang-go
    spec:
      containers:
      - args:
        - -mode=websocket
        - -addr=:7777
        env:
        # ⚠️ Necessary until the buildserver code is moved to go-langserver
        - name: CONFIG_FILE_HASH
          value: 028ee65ade4ca84a16c28f4f91bfc0769d4ce248bc7a6e8a8bc7078e848bf20f
        - name: LIGHTSTEP_ACCESS_TOKEN
          value: '???'
        - name: LIGHTSTEP_INCLUDE_SENSITIVE
          value: "true"
        - name: LIGHTSTEP_PROJECT
          value: sourcegraph-prod
        # ⚠️ Necessary until the buildserver code is moved to go-langserver
        - name: SOURCEGRAPH_CONFIG_FILE
          value: /etc/sourcegraph/config.json
        # TLS is optional
        - name: TLS_CERT
          valueFrom:
            secretKeyRef:
              key: cert
              name: tls
        - name: TLS_KEY
          valueFrom:
            secretKeyRef:
              key: key
              name: tls
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: CACHE_DIR
          value: /mnt/cache/$(POD_NAME)
        image: sourcegraph/xlang-go:latest
        livenessProbe:
          initialDelaySeconds: 5
          tcpSocket:
            port: lsp
          timeoutSeconds: 5
        name: lang-go
        ports:
        - containerPort: 7777
          name: lsp
        - containerPort: 6060
          name: debug
        readinessProbe:
          tcpSocket:
            port: 7777
        resources:
          limits:
            cpu: "8"
            memory: 10G
          requests:
            cpu: "1"
            memory: 10G
        volumeMounts:
        # ⚠️ Necessary until the buildserver code is moved to go-langserver
        - mountPath: /etc/sourcegraph
          name: sg-config
        - mountPath: /mnt/cache
          name: cache-ssd
      volumes:
        # ⚠️ Necessary until the buildserver code is moved to go-langserver
      - configMap:
          defaultMode: 464
          name: config-file
        name: sg-config
      - hostPath:
          path: /mnt/disks/ssd0/pod-tmp
        name: cache-ssd
```

## Settings

From [./src/settings.ts](./src/settings.ts):

```typescript
export interface FullSettings {
    /**
     * The address to the Go language server listening for WebSocket connections.
     */
    'go.serverUrl': string
    /**
     * The key in settings where this extension looks to find the access token
     * for the current user.
     */
    'go.accessToken': string
    /**
     * Whether or not to return external references (from other repositories)
     * along with local references.
     */
    'go.showExternalReferences': boolean
    /**
     * The maximum number of repositories to look in when searching for external
     * references for a symbol (defaults to 50).
     */
    'go.maxExternalReferenceRepos': number
    /**
     * When set, will cause this extension to use to use gddo's (Go Doc Dot Org) API
     * (https://github.com/golang/gddo) to find packages that import a given
     * package (used in finding external references). This cannot be set to
     * `https://godoc.org` because gddo does not set CORS headers. You'll
     * need a proxy to get around this.
     */
    'go.gddoURL': string
}
```

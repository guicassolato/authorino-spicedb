# Demo: Zanzibar-like authorization with SpiceDB and Authorino

This is a demo of integrating Authorino with Authzed's [SpiceDB](https://authzed.com/spicedb).

SpiceDB is a Google [Zanzibar](https://research.google/pubs/pub48190)-inspired authorization system that, like Google Zanzibar, allows for the modeling of fine-grained permissions based on relationships (Relationship-Based Access Control, or ReBAC).

One of the main challenges of implementing fine-grained permissions with an external authorization system is making that system aware of the existing relations. In this demo, we use Authorino [callbacks](https://github.com/Kuadrant/authorino/blob/main/docs/features.md#callbacks-callbacks) to inform SpiceDB about the permissions implied by the operations requested by the users, such as creating or deleting an application resource, as well as granting and revoking access to resources for third-party users.

The full scope of the demo consists of protecting endpoints of a REST API that handles documents, the Docs API. Any authenticated user with a valid API key is allowed to create documents. Users can read and delete their own documents, as well as grant read access to their documents for other users. All fine-grained permissions involved are automatically stored in SpiceDB by Authorino, based on the operations requested by the users to the Docs API.

## Requirements

- [Docker](https://docker.com)
- [Kind](https://kind.sigs.k8s.io)

## The stack

- **Kubernetes cluster**<br/>
  Started locally with [Kind](https://kind.sigs.k8s.io/).
- **Docs API**<br/>
  A REST API application that will be protected using SpiceDB and Authorino.<br/>
  The following HTTP endpoints are available:
  ```
  POST /docs/{id}    Create a doc
  GET /docs          List all docs
  GET /docs/{id}     Read a doc
  DELETE /docs/{id}  Delete a doc
  ```
- **[Envoy proxy](https://envoyproxy.io)**<br/>
  Deployed as sidecar of the Docs API, to serve the application with the [External Authorization](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/ext_authz_filter#config-http-filters-ext-authz) filter enabled and pointing to Authorino. After deploying the sidecar, the following additional endpoints are introduced:
  ```
  POST /docs/{id}/allow/{user}    Grant read access to the doc
  DELETE /docs/{id}/allow/{user}  Revoke read access to the doc
  ```
- **[Authorino Operator](https://github.com/kuadrant/authorino-operator)**<br/>
  Cluster-wide installation of the operator and CRDs to manage and use Authorino authorization services.
- **[Authorino](https://github.com/kuadrant/authorino)**<br/>
  The external authorization service, deployed in [`namespaced`](https://github.com/Kuadrant/authorino/blob/main/docs/architecture.md#cluster-wide-vs-namespaced-instances) reconciliation mode, in the same K8s namespace as the Docs API.
- **[SpiceDB](https://authzed.com/spicedb)**<br/>
  Open source Zanzibar-inspired database to store, compute, and validate fine grained permissions.
- **[Contour](https://projectcontour.io)**<br/>
  Kubernetes Ingress Controller based on the Envoy proxy, to handle the ingress traffic to the Docs API and to Keycloak.

> **Note:** For simplicity, in the demo all components are deployed without TLS.

## Setup

<details>
  <summary>🅰 Create the cluster</summary>

  ```sh
  kind create cluster --name authorino-demo --config -<<EOF
  apiVersion: kind.x-k8s.io/v1alpha4
  kind: Cluster
  nodes:
  - role: control-plane
    extraPortMappings:
    - containerPort: 80
      hostPort: 80
      listenAddress: "0.0.0.0"
    - containerPort: 443
      hostPort: 443
      listenAddress: "0.0.0.0"
  EOF
  ```
</details>

<details>
  <summary>🅱 Install Contour</summary>

  ```sh
  kubectl apply -f https://raw.githubusercontent.com/guicassolato/authorino-spicedb/main/contour.yaml
  ```
</details>

<details>
  <summary>🅲 Install the Authorino Operator</summary>

  ```sh
  kubectl apply -f https://raw.githubusercontent.com/Kuadrant/authorino-operator/main/config/deploy/manifests.yaml

  # TODO: Remove after https://github.com/kuadrant/authorino/pull/375 is merged and the Operator is up to date with the latest version of the manifests
  kubectl apply -f https://raw.githubusercontent.com/Kuadrant/authorino/authzed/install/manifests.yaml
  ```

  > **Note:** In OpenShift, the Authorino Operator can alternatively be installed directly from the Red Hat OperatorHub, using [Operator Lifecycle Manager](https://olm.operatorframework.io/).
</details>

## Run the demo ① → ④

### ① Deploy the Docs API

<details>
  <summary>🅰 Create the namespace</summary>

  ```sh
  kubectl create namespace docs-api
  ```
</details>

<details>
  <summary>🅱 Deploy the Docs API in the namespace</summary>

  ```sh
  kubectl -n docs-api apply -f -<<EOF
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: docs-api
    labels:
      app: docs-api
  spec:
    selector:
      matchLabels:
        app: docs-api
    template:
      metadata:
        labels:
          app: docs-api
      spec:
        containers:
          - name: docs-api
            image: quay.io/kuadrant/authorino-examples:news-api
            imagePullPolicy: IfNotPresent
            env:
              - name: PORT
                value: "3000"
            tty: true
            ports:
              - containerPort: 3000
    replicas: 1
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: docs-api
    labels:
      app: docs-api
  spec:
    selector:
      app: docs-api
    ports:
      - name: http
        port: 3000
        protocol: TCP
  ---
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: docs-api
    labels:
      app: docs-api
  spec:
    rules:
      - host: docs-api.127.0.0.1.nip.io
        http:
          paths:
            - backend:
                service:
                  name: docs-api
                  port:
                    number: 3000
              path: /docs
              pathType: Prefix
  EOF
  ```
</details>

<details>
  <summary>🅲 Try the Docs API unprotected</summary>

  ```sh
  curl http://docs-api.127.0.0.1.nip.io/docs -i
  # HTTP/1.1 200 OK
  ```
</details>

### ② Create the permissions database

<details>
  <summary>🅰 Create the namespace</summary>

  ```sh
  kubectl create namespace spicedb
  ```
</details>

<details>
  <summary>🅱 Deploy the SpiceDB instance</summary>

  ```sh
  kubectl -n spicedb apply -f -<<EOF
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: spicedb
    labels:
      app: spicedb
  spec:
    selector:
      matchLabels:
        app: spicedb
    template:
      metadata:
        labels:
          app: spicedb
      spec:
        containers:
          - name: spicedb
            image: authzed/spicedb
            args:
              - serve
              - "--grpc-preshared-key"
              - secret
              - "--http-enabled"
            ports:
              - containerPort: 50051
              - containerPort: 8443
    replicas: 1
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: spicedb
  spec:
    selector:
      app: spicedb
    ports:
      - name: grpc
        port: 50051
        protocol: TCP
      - name: http
        port: 8443
        protocol: TCP
  ---
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: spicedb
    labels:
      app: spicedb
  spec:
    rules:
      - host: spicedb.127.0.0.1.nip.io
        http:
          paths:
            - backend:
                service:
                  name: spicedb
                  port:
                    number: 8443
              path: /
              pathType: Prefix
  EOF
  ```
</details>

<details>
  <summary>🅲 Create the permission schema</summary>

  ```sh
  curl -X POST http://spicedb.127.0.0.1.nip.io/v1/schema/write \
       -H 'Authorization: Bearer secret' \
       -H 'Content-Type: application/json' \
       -d @- <<EOF
  {
    "schema": "definition user {}\ndefinition doc {\n\trelation reader: user\n\trelation writer: user\n\n\tpermission read = reader + writer\n\tpermission write = writer\n}"
  }
  EOF
  ```
</details>

### ③ Lock down the Docs API

<details>
  <summary>🅰 Request an instance of Authorino</summary>

  ```sh
  kubectl -n docs-api apply -f -<<EOF
  apiVersion: operator.authorino.kuadrant.io/v1beta1
  kind: Authorino
  metadata:
    name: authorino
  spec:
    listener:
      tls:
        enabled: false
    oidcServer:
      tls:
        enabled: false

    # TODO: Remove after https://github.com/kuadrant/authorino/pull/375 is merged
    image: quay.io/kuadrant/authorino:authzed
  EOF
  ```
</details>

<details>
  <summary>🅱 Redeploy the Docs API with the sidecar proxy</summary>

  ```sh
  kubectl -n docs-api apply -f -<<EOF
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: docs-api
    labels:
      app: docs-api
  spec:
    selector:
      matchLabels:
        app: docs-api
    template:
      metadata:
        labels:
          app: docs-api
      spec:
        containers:
          - name: docs-api
            image: quay.io/kuadrant/authorino-examples:news-api
            imagePullPolicy: IfNotPresent
            env:
              - name: PORT
                value: "3000"
            tty: true
            ports:
              - containerPort: 3000
          - name: envoy
            image: envoyproxy/envoy:v1.19-latest
            imagePullPolicy: IfNotPresent
            command:
              - /usr/local/bin/envoy
            args:
              - --config-path /usr/local/etc/envoy/envoy.yaml
              - --service-cluster front-proxy
              - --log-level info
              - --component-log-level filter:trace,http:debug,router:debug
            ports:
              - containerPort: 8000
            volumeMounts:
              - mountPath: /usr/local/etc/envoy
                name: config
                readOnly: true
        volumes:
          - name: config
            configMap:
              items:
                - key: envoy.yaml
                  path: envoy.yaml
              name: envoy
    replicas: 1
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: docs-api
    labels:
      app: docs-api
  spec:
    selector:
      app: docs-api
    ports:
      - name: envoy
        port: 8000
        protocol: TCP
  ---
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: docs-api
    labels:
      app: docs-api
  spec:
    rules:
      - host: docs-api.127.0.0.1.nip.io
        http:
          paths:
            - backend:
                service:
                  name: docs-api
                  port:
                    number: 8000
              path: /docs
              pathType: Prefix
  ---
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: envoy
    labels:
      app: envoy
  data:
    envoy.yaml: |
      static_resources:
        clusters:
          - name: docs-api
            connect_timeout: 0.25s
            type: strict_dns
            lb_policy: round_robin
            load_assignment:
              cluster_name: docs-api
              endpoints:
                - lb_endpoints:
                    - endpoint:
                        address:
                          socket_address:
                            address: 127.0.0.1
                            port_value: 3000
          - name: authorino
            connect_timeout: 0.25s
            type: strict_dns
            lb_policy: round_robin
            http2_protocol_options: {}
            load_assignment:
              cluster_name: authorino
              endpoints:
                - lb_endpoints:
                    - endpoint:
                        address:
                          socket_address:
                            address: authorino-authorino-authorization
                            port_value: 50051
        listeners:
          - address:
              socket_address:
                address: 0.0.0.0
                port_value: 8000
            filter_chains:
              - filters:
                  - name: envoy.http_connection_manager
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                      stat_prefix: local
                      route_config:
                        name: docs-api
                        virtual_hosts:
                          - name: docs-api
                            domains: ['*']
                            routes:
                              - match:
                                  prefix: /
                                route:
                                  cluster: docs-api
                      http_filters:
                        - name: envoy.filters.http.ext_authz
                          typed_config:
                            "@type": type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthz
                            transport_api_version: V3
                            failure_mode_allow: false
                            include_peer_certificate: true
                            grpc_service:
                              envoy_grpc:
                                cluster_name: authorino
                              timeout: 1s
                        - name: envoy.filters.http.lua
                          typed_config:
                            "@type": type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua
                            inline_code: |
                              function envoy_on_request(request_handle)
                                if string.match(request_handle:headers():get(":path"), '^/docs/[^/]+/allow/.+') then
                                  request_handle:respond({[":status"] = "200"}, "")
                                end
                              end
                        - name: envoy.filters.http.router
                          typed_config: {}
                      use_remote_address: true
      admin:
        access_log_path: "/tmp/admin_access.log"
        address:
          socket_address:
            address: 0.0.0.0
            port_value: 8001
  EOF
  ```
</details>

<details>
  <summary>🅲 Try the Docs API without authentication</summary>

  ```sh
  curl http://docs-api.127.0.0.1.nip.io/docs -i
  # HTTP/1.1 404 Not Found
  # x-ext-auth-reason: Service not found
  # server: envoy
  # ...
  ```
</details>

### ④ Open up the Docs API for authenticated and authorized users

<details>
  <summary>🅰 Create the AuthConfig</summary>

  ```sh
  kubectl -n docs-api apply -f -<<EOF
  apiVersion: authorino.kuadrant.io/v1beta1
  kind: AuthConfig
  metadata:
    name: docs-api-protection
  spec:
    hosts:
      - docs-api.127.0.0.1.nip.io

    # Users authenticated with API keys
    identity:
      - name: api-key-users
        apiKey:
          selector:
            matchLabels:
              app: docs-api
        credentials:
          in: authorization_header
          keySelector: APIKEY

    metadata:
      # List resources → lokkup resources the user has access to
      - name: permission-lookup
        when:
          - selector: context.request.http.method
            operator: eq
            value: GET
          - selector: context.request.http.path.@extract:{"sep":"/","pos":2}
            operator: eq
            value: ""
        http:
          endpoint: http://spicedb.spicedb.svc.cluster.local:8443/v1/permissions/resources
          method: POST
          contentType: application/json
          body:
            valueFrom:
              authJSON: |
                \{
                  "resourceObjectType":"doc",
                  "permission":"read",
                  "subject":\{
                    "object":\{
                      "objectType":"user",
                      "objectId":"{auth.identity.metadata.annotations.username}"
                    \}
                  \}
                \}
          sharedSecretRef:
            name: spicedb
            key: token

    authorization:
      # Read or delete a resource → check in SpiceDB if the user has read or write permission respectively
      - name: read-or-delete-resource
        when:
          - selector: context.request.http.method
            operator: neq
            value: POST
          - selector: context.request.http.path.@extract:{"sep":"/","pos":2}
            operator: neq
            value: ""
          - selector: context.request.http.path.@extract:{"sep":"/","pos":3}
            operator: neq
            value: allow
        authzed:
          endpoint: spicedb.spicedb.svc.cluster.local:50051
          insecure: true
          sharedSecretRef:
            name: spicedb
            key: token
          subject:
            kind:
              value: user
            name:
              valueFrom:
                authJSON: auth.identity.metadata.annotations.username
          resource:
            kind:
              value: doc
            name:
              valueFrom:
                authJSON: context.request.http.path.@extract:{"sep":"/","pos":2}
          permission:
            valueFrom:
              authJSON: context.request.http.method.@replace:{"old":"GET","new":"read"}.@replace:{"old":"DELETE","new":"write"}

      # Grant or revoke access to resource → check in SpiceDB if the user has write permission
      - name: grant-or-revoke-access-to-resource
        when:
          - selector: context.request.http.path.@extract:{"sep":"/","pos":3}
            operator: eq
            value: allow
        authzed:
          endpoint: spicedb.spicedb.svc.cluster.local:50051
          insecure: true
          sharedSecretRef:
            name: spicedb
            key: token
          subject:
            kind:
              value: user
            name:
              valueFrom:
                authJSON: auth.identity.metadata.annotations.username
          resource:
            kind:
              value: doc
            name:
              valueFrom:
                authJSON: context.request.http.path.@extract:{"sep":"/","pos":2}
          permission:
            value: write

    response:
      # Create new resource → inject user info in the request
      - name: x-ext-auth-data
        when:
          - selector: context.request.http.method
            operator: eq
            value: POST
          - selector: context.request.http.path.@extract:{"sep":"/","pos":3}
            operator: neq
            value: allow
        json:
          properties:
            - name: author
              valueFrom: { authJSON: auth.identity.metadata.annotations.fullname }
            - name: user_id
              valueFrom: { authJSON: auth.identity.metadata.annotations.username }

      # List resources → filter resource ids the user has access to
      - name: x-filter
        when:
          - selector: context.request.http.method
            operator: eq
            value: GET
          - selector: context.request.http.path.@extract:{"sep":"/","pos":2}
            operator: eq
            value: ""
        json:
          properties:
            - name: id
              valueFrom:
                authJSON: auth.metadata.permission-lookup.result.resourceObjectId
            - name: ids
              valueFrom:
                authJSON: auth.metadata.permission-lookup.#.result.resourceObjectId

    callbacks:
      # Create new resource → create 'writer' relationship in SpiceDB
      - name: create-resource
        when:
          - selector: context.request.http.method
            operator: eq
            value: POST
          - selector: context.request.http.path.@extract:{"sep":"/","pos":3}
            operator: neq
            value: allow
        http:
          endpoint: http://spicedb.spicedb.svc.cluster.local:8443/v1/relationships/write
          method: POST
          contentType: application/json
          body:
            valueFrom:
              authJSON: |
                \{
                  "updates":[
                    \{
                      "operation":"OPERATION_CREATE",
                      "relationship":\{
                        "resource":\{
                          "objectType":"doc",
                          "objectId":"{context.request.http.path.@extract:{"sep":"/","pos":2}}"
                        \},
                        "relation":"writer",
                        "subject":\{
                          "object":\{
                            "objectType":"user",
                            "objectId":"{auth.identity.metadata.annotations.username}"
                          \}
                        \}
                      \}
                    \}
                  ]
                \}
          sharedSecretRef:
            name: spicedb
            key: token

      # Delete resource → delete all corresponding relationships in SpiceDB
      - name: delete-resource
        when:
          - selector: context.request.http.method
            operator: eq
            value: DELETE
          - selector: auth.authorization.spicedb
            operator: neq
            value: ""
        http:
          endpoint: http://spicedb.spicedb.svc.cluster.local:8443/v1/relationships/delete
          method: POST
          contentType: application/json
          body:
            valueFrom:
              authJSON: |
                \{
                  "relationshipFilter": \{
                    "resourceType": "doc",
                    "optionalResourceId": "{context.request.http.path.@extract:{"sep":"/","pos":2}}"
                  \}
                \}
          sharedSecretRef:
            name: spicedb
            key: token

      # Grant access to resource → create 'reader' relationship in SpiceDB
      - name: grant-access
        when:
          - selector: context.request.http.method
            operator: eq
            value: POST
          - selector: context.request.http.path.@extract:{"sep":"/","pos":3}
            operator: eq
            value: allow
        http:
          endpoint: http://spicedb.spicedb.svc.cluster.local:8443/v1/relationships/write
          method: POST
          contentType: application/json
          body:
            valueFrom:
              authJSON: |
                \{
                  "updates":[
                    \{
                      "operation":"OPERATION_CREATE",
                      "relationship":\{
                        "resource":\{
                          "objectType":"doc",
                          "objectId":"{context.request.http.path.@extract:{"sep":"/","pos":2}}"
                        \},
                        "relation":"reader",
                        "subject":\{
                          "object":\{
                            "objectType":"user",
                            "objectId":"{context.request.http.path.@extract:{"sep":"/","pos":4}}"
                          \}
                        \}
                      \}
                    \}
                  ]
                \}
          sharedSecretRef:
            name: spicedb
            key: token

      # Revoke access to resource → delete 'reader' relationships in SpiceDB
      - name: revoke-access
        when:
          - selector: context.request.http.method
            operator: eq
            value: DELETE
          - selector: context.request.http.path.@extract:{"sep":"/","pos":3}
            operator: eq
            value: allow
        http:
          endpoint: http://spicedb.spicedb.svc.cluster.local:8443/v1/relationships/delete
          method: POST
          contentType: application/json
          body:
            valueFrom:
              authJSON: |
                \{
                  "relationshipFilter": \{
                    "resourceType": "doc",
                    "optionalResourceId": "{context.request.http.path.@extract:{"sep":"/","pos":2}}",
                    "optionalRelation": "reader",
                    "optionalSubjectFilter": \{
                      "subjectType": "user",
                      "optionalSubjectId": "{context.request.http.path.@extract:{"sep":"/","pos":4}}"
                    \}
                  \}
                \}
          sharedSecretRef:
            name: spicedb
            key: token
  ---
  apiVersion: v1
  kind: Secret
  metadata:
    name: spicedb
    labels:
      app: spicedb
  stringData:
    token: secret
  EOF
  ```
</details>

<details>
  <summary>🅱 Create the API keys for users to consume the Docs API</summary>

  ```sh
  kubectl -n docs-api apply -f -<<EOF
  apiVersion: v1
  kind: Secret
  metadata:
    name: api-key-writer
    labels:
      authorino.kuadrant.io/managed-by: authorino
      app: docs-api
    annotations:
      username: emilia
      fullname: 👩🏾 Emilia Jones
  stringData:
    api_key: IAMEMILIA
  ---
  apiVersion: v1
  kind: Secret
  metadata:
    name: api-key-reader
    labels:
      authorino.kuadrant.io/managed-by: authorino
      app: docs-api
    annotations:
      username: beatrice
      fullname: 🧑🏻‍🦰 Beatrice Smith
  stringData:
    api_key: IAMBEATRICE
  EOF
  ```
</details>

<details>
  <summary>🅲 Consume the Docs API fully protected</summary>

  <br/>

  As 👩🏾 Emilia, **create** a doc:

  ```sh
  curl -H 'Authorization: APIKEY IAMEMILIA' \
     -X POST \
     -H 'Content-Type: application/json' \
     -d '{"title":"Emilia´s doc","body":"This is Emilia´s doc."}' \
     http://docs-api.127.0.0.1.nip.io/docs/e9ebb594-c3fc-4f0d-bbbd-a0fd3fac6639 -i
  # HTTP/1.1 200 OK
  # ...
  # {"id":"e9ebb594-c3fc-4f0d-bbbd-a0fd3fac6639","title":"Emilia´s doc","body":"This is Emilia´s doc.","date":"2023-02-07 18:17:30 +0000","author":"👩🏾 Emilia Jones","user_id":"emilia"}
  ```

  As 👩🏾 Emilia, **read** the doc just created:

  ```sh
  curl -H 'Authorization: APIKEY IAMEMILIA' \
     -X GET \
     http://docs-api.127.0.0.1.nip.io/docs/e9ebb594-c3fc-4f0d-bbbd-a0fd3fac6639 -i
  # HTTP/1.1 200 OK
  ```

  As 🧑🏻‍🦰 Beatrice, try to **read** the doc created by Emilia:

  ```sh
  curl -H 'Authorization: APIKEY IAMBEATRICE' \
     -X GET \
     http://docs-api.127.0.0.1.nip.io/docs/e9ebb594-c3fc-4f0d-bbbd-a0fd3fac6639 -i
  # HTTP/1.1 403 Forbidden
  # x-ext-auth-reason: PERMISSIONSHIP_NO_PERMISSION;token=...
  ```

  As 👩🏾 Emilia, **grant** access to the doc for 🧑🏻‍🦰 Beatrice:

  ```sh
  curl -H 'Authorization: APIKEY IAMEMILIA' \
     -X POST \
     http://docs-api.127.0.0.1.nip.io/docs/e9ebb594-c3fc-4f0d-bbbd-a0fd3fac6639/allow/beatrice -i
  # HTTP/1.1 200 OK
  ```

  As 🧑🏻‍🦰 Beatrice, try again to **read** the doc owned by Emilia:

  ```sh
  curl -H 'Authorization: APIKEY IAMBEATRICE' \
     -X GET \
     http://docs-api.127.0.0.1.nip.io/docs/e9ebb594-c3fc-4f0d-bbbd-a0fd3fac6639 -i
  # HTTP/1.1 200 OK
  ```

  As 🧑🏻‍🦰 Beatrice, **create** a doc of her own:

  ```sh
  curl -H 'Authorization: APIKEY IAMBEATRICE' \
     -X POST \
     -H 'Content-Type: application/json' \
     -d '{"title":"Beatrice´s doc","body":"This is Beatrice´s doc."}' \
     http://docs-api.127.0.0.1.nip.io/docs/eed6a74b-ccb1-4e8f-afab-be2a5e1bd97b -i
  # HTTP/1.1 200 OK
  # ...
  # {"id":"eed6a74b-ccb1-4e8f-afab-be2a5e1bd97b","title":"Beatrice´s doc","body":"This is Beatrice´s doc.","date":"2023-02-07 18:25:10 +0000","author":"🧑🏻‍🦰 Beatrice Smith","user_id":"beatrice"}
  ```

  As 🧑🏻‍🦰 Beatrice, **list** all the docs Beatrice has access to:

  ```sh
  curl -H 'Authorization: APIKEY IAMBEATRICE' \
     http://docs-api.127.0.0.1.nip.io/docs -i
  # HTTP/1.1 200 OK
  # ...
  # [
  #   {"id":"e9ebb594-c3fc-4f0d-bbbd-a0fd3fac6639","title":"Emilia´s doc","body":"This is Emilia´s doc.","date":"2023-02-07 18:17:30 +0000","author":"👩🏾 Emilia Jones","user_id":"emilia"},
  #   {"id":"eed6a74b-ccb1-4e8f-afab-be2a5e1bd97b","title":"Beatrice´s doc","body":"This is Beatrice´s doc.","date":"2023-02-07 18:25:10 +0000","author":"🧑🏻‍🦰 Beatrice Smith","user_id":"beatrice"}
  # ]
  ```

  As 👩🏾 Emilia, **list** all the docs Emilia has access to:

  ```sh
  curl -H 'Authorization: APIKEY IAMEMILIA' \
     http://docs-api.127.0.0.1.nip.io/docs -i
  # HTTP/1.1 200 OK
  # ...
  # [{"id":"e9ebb594-c3fc-4f0d-bbbd-a0fd3fac6639","title":"Emilia´s doc","body":"This is Emilia´s doc.","date":"2023-02-07 18:17:30 +0000","author":"👩🏾 Emilia Jones","user_id":"emilia"}]
  ```

  As 👩🏾 Emilia, **revoke** 🧑🏻‍🦰 Beatrice's access to the doc:

  ```sh
  curl -H 'Authorization: APIKEY IAMEMILIA' \
     -X DELETE \
     http://docs-api.127.0.0.1.nip.io/docs/e9ebb594-c3fc-4f0d-bbbd-a0fd3fac6639/allow/beatrice -i
  # HTTP/1.1 200 OK
  ```

  As 🧑🏻‍🦰 Beatrice, **list** again the docs Beatrice has access to:

  ```sh
  curl -H 'Authorization: APIKEY IAMBEATRICE' \
     http://docs-api.127.0.0.1.nip.io/docs -i
  # HTTP/1.1 200 OK
  # ...
  # [{"id":"eed6a74b-ccb1-4e8f-afab-be2a5e1bd97b","title":"Beatrice´s doc","body":"This is Beatrice´s doc.","date":"2023-02-07 18:25:10 +0000","author":"🧑🏻‍🦰 Beatrice Smith","user_id":"beatrice"}]
  ```

  As 🧑🏻‍🦰 Beatrice, try one last time to **read** the doc owned by Emilia:

  ```sh
  curl -H 'Authorization: APIKEY IAMBEATRICE' \
     -X GET \
     http://docs-api.127.0.0.1.nip.io/docs/e9ebb594-c3fc-4f0d-bbbd-a0fd3fac6639 -i
  # HTTP/1.1 403 Forbidden
  # x-ext-auth-reason: PERMISSIONSHIP_NO_PERMISSION;token=...
  ```

  As 👩🏾 Emilia, **delete** the doc:

  ```sh
  curl -H 'Authorization: APIKEY IAMEMILIA' \
     -X DELETE \
     http://docs-api.127.0.0.1.nip.io/docs/e9ebb594-c3fc-4f0d-bbbd-a0fd3fac6639 -i
  # HTTP/1.1 200 OK
  ```

  As 👩🏾 Emilia, retry to **read** the doc just deleted:

  ```sh
  curl -H 'Authorization: APIKEY IAMEMILIA' \
     -X GET \
     http://docs-api.127.0.0.1.nip.io/docs/e9ebb594-c3fc-4f0d-bbbd-a0fd3fac6639 -i
  # HTTP/1.1 404 Not Found
  ```
</details>

## Cleanup

```sh
kind delete cluster --name authorino-demo
```

## Caveats

### Consistency

Because Authorino builds in SpiceDB the permission relationships implied by the requests sent to the Docs API _before_ these requests effectively hit the application, and at the same time the application itself has no knowledge of the authorization system in place at all, the system may run into a situation where the resources and relations stored in the application mismatch the state of the relationships in SpiceDB. This can happen, for example, if an authorized request (i.e. after passing Authorino) fails to be processed by the application due to a server error.

To mitigate the risk of consistency issues, the HTTP requests sent to the Docs API must be treated as an atomic transaction from the moment Envoy hands it over to Authorino, until the upstream application response is processed by Envoy.

To be able to recover from possible consistency issues in case the mitigation fails, logs of the requests handled by Authorino can be stored including timestamp, username, as well as method and path of the HTTP request. Such logs can be implemented by adding another Authorino [callback](https://github.com/Kuadrant/authorino/blob/main/docs/features.md#callbacks-callbacks) in the AuthConfig. The system should occasionally check for consistency issues and use the logs to rebuild the desired state incrementally.

### Latency

Compared to monolithic approach of embedded authorization rules and without proxy in the middle, another caveat of this implementation comes from the extra hops involved in the communication between sidecar proxy (Envoy) and authorization service (Authorino), authorization service and permission database/policy engine (SpiceDB), and sidecar proxy and upstream application (Docs API), and its significance in terms of added latency to the overall request.

To mitigate the impact on latency especially due to the HTTP and GRPC communication between Authorino and SpiceDB, [caching](https://github.com/Kuadrant/authorino/blob/main/docs/features.md#common-feature-caching-cache) can be enabled in Authorino for the `metadata` and `authorization` rules.

Performance can also be improved once `callbacks` in the AuthConfig can be processed asynchronously (see https://github.com/Kuadrant/authorino/issues/369).

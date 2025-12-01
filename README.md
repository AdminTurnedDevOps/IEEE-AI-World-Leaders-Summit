## Gateway Setup

1. Create env variable for Anthropic key

```
export ANTHROPIC_API_KEY=
```

2. Create a Gateway for Anthropic

A `Gateway` resource is used to trigger kgateway to deploy agentgateway data plane Pods

The Agentgateway data plane Pod is the Pod that gets created when a Gateway object is created in a Kubernetes environment where Agentgateway is deployed as the Gateway API implementation.
```
kubectl apply -f- <<EOF
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: agentgateway
  namespace: gloo-system
  labels:
    app: agentgateway
spec:
  gatewayClassName: agentgateway-enterprise
  listeners:
  - protocol: HTTP
    port: 8080
    name: http
    allowedRoutes:
      namespaces:
        from: All
EOF
```

3. Create a secret to store the Claude API key
```
kubectl apply -f- <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: anthropic-secret
  namespace: gloo-system
  labels:
    app: agentgateway
type: Opaque
stringData:
  Authorization: $ANTHROPIC_API_KEY
EOF
```

4. Create a `Backend` object 

A Backend resource to define a backing destination that you want kgateway to route to. In this case, it's Claude.
```
kubectl apply -f- <<EOF
apiVersion: gateway.kgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  labels:
    app: agentgateway
  name: anthropic
  namespace: gloo-system
spec:
  ai:
    provider:
        anthropic:
          model: "claude-3-5-haiku-latest"
  policies:
    auth:
      secretRef:
        name: anthropic-secret
EOF
```

5. Ensure everything is running as expected
```
kubectl get agentgatewaybackend -n gloo-system
```

6. Apply the Route so you can reach the LLM
```
kubectl apply -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: claude
  namespace: gloo-system
  labels:
    app: agentgateway
spec:
  parentRefs:
    - name: agentgateway
      namespace: gloo-system
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /anthropic
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplaceFullPath
          replaceFullPath: /v1/chat/completions
    backendRefs:
    - name: anthropic
      namespace: gloo-system
      group: gateway.kgateway.dev
      kind: AgentgatewayBackend
EOF
```

7. Capture the LB IP of the service. This will be used later to send a request to the LLM.
```
export INGRESS_GW_ADDRESS=$(kubectl get svc -n gloo-system agentgateway -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")
echo $INGRESS_GW_ADDRESS
```

8. Test the LLM connectivity
```
curl "$INGRESS_GW_ADDRESS:8080/anthropic" -H content-type:application/json -H x-api-key:$ANTHROPIC_API_KEY -H "anthropic-version: 2023-06-01" -d '{
  "messages": [
    {
      "role": "system",
      "content": "You are a skilled cloud-native network engineer."
    },
    {
      "role": "user",
      "content": "What is a credit card?"
    }
  ]
}' | jq
```

## MCP Server Connection

1. . Deploy the MCP Server
```
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-website-fetcher
  namespace: default
spec:
  selector:
    matchLabels:
      app: mcp-website-fetcher
  template:
    metadata:
      labels:
        app: mcp-website-fetcher
    spec:
      containers:
      - name: mcp-website-fetcher
        image: ghcr.io/peterj/mcp-website-fetcher:main
        imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: mcp-website-fetcher
  namespace: default
spec:
  selector:
    app: mcp-website-fetcher
  ports:
  - port: 80
    targetPort: 8000
    appProtocol: kgateway.dev/mcp
EOF
```

2. Create the Backend
```
kubectl apply -f - <<EOF
apiVersion: gateway.kgateway.dev/v1alpha1
kind: Backend
metadata:
  name: mcp-backend
  namespace: kgateway-system
spec:
  type: MCP
  mcp:
    targets:
    - name: mcp-target
      static:
        host: mcp-website-fetcher.default.svc.cluster.local
        port: 80
        protocol: SSE
EOF
```

3. Create the Gateway
```
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: agentgateway
  namespace: kgateway-system
spec:
  gatewayClassName: agentgateway
  listeners:
  - name: http
    port: 8080
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same
EOF
```

4. Create the HTTPRoute
```
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mcp-route
  namespace: kgateway-system
spec:
  parentRefs:
  - name: agentgateway
  rules:
  - backendRefs:
    - name: mcp-backend
      group: gateway.kgateway.dev
      kind: Backend
EOF
```

5. Test the Gateway/route
```
export GATEWAY_IP=$(kubectl get svc agentgateway -n kgateway-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $GATEWAY_IP


curl -v "http://${GATEWAY_IP}:8080/" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc": "2.0",
    "method": "initialize",
    "params": {
      "protocolVersion": "2024-11-05",
      "capabilities": {},
      "clientInfo": {
        "name": "test-client",
        "version": "1.0.0"
      }
    },
    "id": 1
  }'
```

## Connecting To Remote MCP Servers

The config below works very much like a package/library import in application code.

When you apply this `MCPServer` resource to your Kubernetes cluster via kagent:

1. kagent (the Kubernetes agent) sees the MCPServer resource
2. It automatically creates a deployment that runs the GitHub MCP Server Docker container
3. Docker pulls the latest version of `ghcr.io/github/github-mcp-server` from GitHub Container Registry
4. The MCP server starts running in a Pod and becomes available via the stdio transport

kagent handles the "package resolution" and deployment automatically. You just declare what MCP server you want (like declaring a dependency in `package.json` or `requirements.txt`), and kagent takes care of fetching, installing, and running it in your cluster.

This is the power of the declarative approach. You specify what you want (the GitHub MCP Server), and kagent figures out how to get it running.

**Note:** The GitHub MCP Server requires a Personal Access Token (PAT) for authentication. You'll need to create a Kubernetes secret with your GitHub PAT before deploying.

Don't specify port configurations since stdio transport doesn't use HTTP

1. Create the github pat token environment variable:
```
export GITHUB_PERSONAL_ACCESS_TOKEN=your_github_pat_here
```

2. Create the k8s secret to store the PAT token:
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: github-pat
  namespace: kagent
type: Opaque
stringData:
  GITHUB_PERSONAL_ACCESS_TOKEN: $GITHUB_PERSONAL_ACCESS_TOKEN
EOF
```

3. Create a new MCP Server object in Kubernetes:
```
kubectl apply -f - <<EOF
apiVersion: kagent.dev/v1alpha2
kind: RemoteMCPServer
metadata:
  name: github-mcp-remote
  namespace: kagent
spec:
  description: GitHub Copilot MCP Server
  url: https://api.githubcopilot.com/mcp/
  protocol: STREAMABLE_HTTP
  headersFrom:
    - name: Authorization
      valueFrom:
        type: Secret
        name: github-pat
        key: GITHUB_PERSONAL_ACCESS_TOKEN
  timeout: 5s
  terminateOnClose: true
EOF
```

4. Create a new Agent and use the MCP Server you created in the previous step:
```
kubectl apply -f - <<EOF
apiVersion: kagent.dev/v1alpha2
kind: Agent
metadata:
  name: github-mcp-agent
  namespace: kagent
spec:
  description: This agent can interact with GitHub repositories, issues, pull requests, and more
  type: Declarative
  declarative:
    modelConfig: default-model-config
    systemMessage: |-
      You're a friendly and helpful agent that uses GitHub tools to help with repository management, issues, pull requests, and code review

      # Instructions

      - If user question is unclear, ask for clarification before running any tools
      - Always be helpful and friendly
      - If you don't know how to answer the question DO NOT make things up
        respond with "Sorry, I don't know how to answer that" and ask the user to further clarify the question

      # Response format
      - ALWAYS format your response as Markdown
      - Your response will include a summary of actions you took and an explanation of the result
    tools:
    - type: McpServer
      mcpServer:
        name: github-mcp-remote
        kind: RemoteMCPServer
        toolNames:
        - get_latest_release
        - get_commit
        - get_tag
        - list_branches
EOF
```

4. Look at the Agent configuration and wait until both `READY` and `ACCEPTED` are in a `True` status:
```
kubectl get agents -n kagent
```

5. Open the kagent UI, go to the UI, and ask: `What branches are available under `https://github.com/AdminTurnedDevOps/agentic-demo-repo``

6. Feel free to dive into how it looks "underneath the hood":
```
kubectl describe agent github-mcp-agent -n kagent
```

### Creating Your GitHub Personal Access Token

To use the GitHub MCP Server, you need a Personal Access Token with appropriate permissions:

1. Go to GitHub Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Click "Generate new token (classic)"
3. Give it a descriptive name like "kagent-mcp-server"
4. Select the following scopes at minimum:
   - `repo` - Full control of private repositories
   - `read:org` - Read org and team membership
   - `read:packages` - Download packages from GitHub Package Registry
5. Click "Generate token" and copy the token immediately (you won't be able to see it again)

For GitHub Enterprise Server, you'll also need to set the `GITHUB_HOST` environment variable in the MCPServer spec.
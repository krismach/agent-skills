# Listen on 0.0.0.0 in Containers

## Section

Deployment & Infrastructure

## Summary

Fastify defaults to 127.0.0.1, which is unreachable from Kubernetes probes. Listen on 0.0.0.0 in containers.

## Incorrect

```javascript
fastify.listen({ port: 3000 })
// Defaults to 127.0.0.1 - unreachable in Kubernetes
```

## Correct

```javascript
fastify.listen({
  port: 3000,
  host: '0.0.0.0' // Reachable from Kubernetes probes
})
```

## Kubernetes Probe Configuration

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 10
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 5
```

## Why

Container orchestration systems need to connect to your application from outside the container. Listening on 0.0.0.0 allows this.

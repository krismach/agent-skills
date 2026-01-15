# Use Reverse Proxy

## Section

Deployment & Infrastructure

## Summary

Never expose Fastify directly to the internet. Use a reverse proxy like Nginx or HAProxy.

## Incorrect

```javascript
const fastify = require('fastify')({
  https: {
    key: fs.readFileSync('server.key'),
    cert: fs.readFileSync('server.cert')
  }
})

fastify.listen({ port: 443, host: '0.0.0.0' })
```

## Correct

```javascript
const fastify = require('fastify')({
  trustProxy: true // Trust X-Forwarded-* headers
})

// Listen on localhost, let proxy handle HTTPS
fastify.listen({ port: 3000, host: '127.0.0.1' })
```

## Nginx Configuration

```nginx
server {
  listen 80;
  listen 443 ssl;
  server_name example.com;

  ssl_certificate /path/to/cert.pem;
  ssl_certificate_key /path/to/key.pem;

  location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

## Why

Reverse proxies handle TLS, load balancing, and static assets more efficiently. They also provide an extra layer of security.

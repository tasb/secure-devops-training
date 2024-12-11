# Demo 06 - Container Vulnerability

## Trivy Dockerfile

- Go to Dockerfile and run trivy command to scan Dockerfile

```bash
trivy config Dockerfile
```

## Trivy Image

- Use created image and run trivy command to scan image

```bash
docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image todo-api:trivy
```

## Show Snyk Integration with VS Code

- Show Snyk extension in VS Code
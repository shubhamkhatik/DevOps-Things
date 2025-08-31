
# Adding `known_hosts` in CI/CD (GitHub Actions)

The `known_hosts` file stores SSH server fingerprints.  
In CI/CD, we need it to avoid interactive "Are you sure you want to continue connecting?" prompts when using SSH.

---

## Method 1: Using a file from the repo (cp)

- Commit your `known_hosts` file into the repo (e.g. at `.cicd/known_hosts`).
- In your workflow, copy it into place:

```yaml
- name: Setup SSH known hosts (from repo)
  run: |
    mkdir -p ~/.ssh
    cp .cicd/known_hosts ~/.ssh/known_hosts
```

✅ Simple, works if you don’t mind the fingerprint stored in repo.
⚠️ Anyone with repo access can see the fingerprint.

## Method 2: Using GitHub Secrets (echo)

Generate fingerprint locally:

ssh-keyscan -H your-server.com


Copy the output (e.g. your-server.com ssh-ed25519 AAAAC3NzaC...).

Save it as a GitHub Secret → KNOWN_HOSTS.

In your workflow:
```
- name: Setup SSH known hosts (from secret)
  run: |
    mkdir -p ~/.ssh
    echo "${{ secrets.KNOWN_HOSTS }}" > ~/.ssh/known_hosts
```

✅ Safer, no fingerprint stored in repo.
⚠️ Need to manage secrets in GitHub.


## Method 3: Pre-populate Known Hosts, direct run keyscan in cicd
```yaml
# GitHub Actions example
- name: Setup SSH
  run: |
    mkdir -p ~/.ssh
    ssh-keyscan -H production-server.com >> ~/.ssh/known_hosts
    chmod 600 ~/.ssh/known_hosts

```


## Method 4: Docker-based Approach

```dockerfile
FROM alpine:latest
RUN apk add --no-cache openssh-client
COPY known_hosts /root/.ssh/known_hosts
RUN chmod 600 /root/.ssh/known_hosts
```
## Where: Platform-Specific Implementations

### GitHub Actions
```yaml
name: Deploy
on: [push]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Setup SSH
      run: |
        mkdir -p ~/.ssh
        echo "${{ secrets.SSH_KNOWN_HOSTS }}" > ~/.ssh/known_hosts
        chmod 600 ~/.ssh/known_hosts
    - name: Deploy
      run: ssh user@server 'deployment-script.sh'
```

### GitLab CI
```yaml
deploy:
  stage: deploy
  before_script:
    - 'command -v ssh-agent >/dev/null || ( apt-get update -y && apt-get install openssh-client -y )'
    - eval $(ssh-agent -s)
    - mkdir -p ~/.ssh
    - echo "$SSH_KNOWN_HOSTS" >> ~/.ssh/known_hosts
    - chmod 644 ~/.ssh/known_hosts
```

### Jenkins Pipeline
```groovy
pipeline {
    agent any
    stages {
        stage('Setup SSH') {
            steps {
                sh '''
                    mkdir -p ~/.ssh
                    echo "${SSH_KNOWN_HOSTS}" > ~/.ssh/known_hosts
                    chmod 600 ~/.ssh/known_hosts
                '''
            }
        }
    }
}
```

### Azure DevOps
```yaml
- task: Bash@3
  displayName: 'Setup SSH Known Hosts'
  inputs:
    targetType: 'inline'
    script: |
      mkdir -p ~/.ssh
      echo "$(SSH_KNOWN_HOSTS)" >> ~/.ssh/known_hosts
      chmod 600 ~/.ssh/known_hosts
```


---

## Edge Cases and Solutions

### Key Rotation Scenarios
```bash
# Handle key rotation gracefully
ssh-keygen -R hostname  # Remove old key
ssh-keyscan -H hostname >> ~/.ssh/known_hosts  # Add new key
```

### Multiple Environments
```yaml
# Environment-specific known hosts
- name: Setup known hosts
  run: |
    case "$ENVIRONMENT" in
      "prod") echo "$PROD_SSH_KNOWN_HOSTS" >> ~/.ssh/known_hosts ;;
      "staging") echo "$STAGING_SSH_KNOWN_HOSTS" >> ~/.ssh/known_hosts ;;
    esac
```

### Container Persistence
```yaml
# Persist across container restarts
volumes:
  - ssh-data:/root/.ssh
```

### Network Issues
```bash
# Fallback mechanism
if ! ssh-keyscan -T 10 hostname >> ~/.ssh/known_hosts; then
    echo "Warning: Could not scan host, using stored backup"
    echo "$BACKUP_HOST_KEY" >> ~/.ssh/known_hosts
fi
```


## Real-World Example: Production Deployment Pipeline

```yaml
name: Production Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Configure SSH
      run: |
        mkdir -p ~/.ssh
        
        # Add known hosts from secrets
        echo "${{ secrets.PROD_SSH_KNOWN_HOSTS }}" >> ~/.ssh/known_hosts
        
        # Verify the format
        if ! ssh-keygen -l -f ~/.ssh/known_hosts; then
          echo "Error: Invalid known_hosts format"
          exit 1
        fi
        
        # Set secure permissions
        chmod 600 ~/.ssh/known_hosts
        
        # Log for debugging (without exposing keys)
        echo "Configured $(wc -l < ~/.ssh/known_hosts) known hosts"
    
    - name: Deploy Application
      run: |
        ssh deploy-user@prod-server.com 'bash -s' < ./deploy.sh
```



















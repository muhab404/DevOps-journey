# GitHub Actions - Complete Study Guide

## 1. Introduction & Basics

**What is GitHub Actions?**
GitHub Actions is a CI/CD (Continuous Integration/Continuous Deployment) platform that allows you to automate workflows directly within your GitHub repository. It enables you to build, test, and deploy code automatically based on events like pushes, pull requests, or scheduled triggers.

**Why is it used?**
- **Automation**: Eliminates manual deployment and testing processes
- **Integration**: Native integration with GitHub repositories
- **Cost-effective**: Free tier with generous limits for public repos
- **Flexibility**: Supports multiple languages, platforms, and cloud providers
- **Community**: Large marketplace of pre-built actions
- **Scalability**: Runs on GitHub-hosted or self-hosted runners

## 2. Core Concepts

### Key Terminology

- **Workflow**: Automated process defined by YAML file in `.github/workflows/`
- **Job**: Set of steps that execute on the same runner
- **Step**: Individual task within a job (run command or action)
- **Action**: Reusable unit of code (custom or from marketplace)
- **Runner**: Server that executes workflows (GitHub-hosted or self-hosted)
- **Event**: Trigger that starts a workflow (push, PR, schedule, etc.)
- **Artifact**: Files produced by workflows that persist between jobs

### Architecture

```
Repository Event → Workflow Trigger → Runner Assignment → Job Execution → Step Processing → Results
```

### Workflow Structure

```yaml
name: Workflow Name
on: [events]
jobs:
  job-name:
    runs-on: runner-type
    steps:
      - name: Step name
        uses: action@version
      - name: Run command
        run: command
```

## 3. Installation/Setup

### Prerequisites
- GitHub repository
- Basic YAML knowledge
- Understanding of your project's build/deploy process

### Setup Steps

1. **Create workflow directory**:
   ```
   mkdir -p .github/workflows
   ```

2. **Create workflow file**:
   ```yaml
   # .github/workflows/ci.yml
   name: CI
   on: [push, pull_request]
   jobs:
     test:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         - name: Run tests
           run: echo "Running tests"
   ```

3. **Commit and push**:
   ```bash
   git add .github/workflows/ci.yml
   git commit -m "Add CI workflow"
   git push
   ```

### Environment Setup
- **Secrets**: Store in Repository Settings → Secrets and variables → Actions
- **Environment variables**: Define in workflow or repository settings
- **Self-hosted runners**: Configure in Settings → Actions → Runners
## 4. Hands-on Examples

### Basic CI Workflow
```yaml
name: CI Pipeline
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
      
      - name: Run linting
        run: npm run lint
```

### Multi-Job Workflow with Dependencies
```yaml
name: Build and Deploy
on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: npm test

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build application
        run: npm run build
      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-files
          path: dist/

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Download artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-files
      - name: Deploy to AWS
        run: aws s3 sync . s3://my-bucket
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

### Matrix Strategy
```yaml
name: Cross-platform Testing
on: [push]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: [16, 18, 20]
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm test
```

## 5. Best Practices

### Security
- **Use secrets for sensitive data**: Never hardcode credentials
- **Pin action versions**: Use specific versions (`@v4`) not tags (`@main`)
- **Limit permissions**: Use `permissions` key to restrict GITHUB_TOKEN
- **Review third-party actions**: Audit marketplace actions before use
- **Use environments**: Protect production deployments with approval gates

```yaml
permissions:
  contents: read
  pull-requests: write
```

### Performance & Scalability
- **Cache dependencies**: Use caching actions to speed up builds
- **Parallel jobs**: Run independent jobs concurrently
- **Conditional execution**: Skip unnecessary steps with `if` conditions
- **Self-hosted runners**: For better performance and cost control

```yaml
- name: Cache dependencies
  uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
```

### Maintainability
- **Reusable workflows**: Create composite actions for repeated logic
- **Clear naming**: Use descriptive names for workflows, jobs, and steps
- **Documentation**: Comment complex workflows
- **Modular design**: Break large workflows into smaller, focused ones
## 6. Common Issues & Troubleshooting

### Frequent Problems

1. **Workflow not triggering**
   - Check file location (`.github/workflows/`)
   - Verify YAML syntax
   - Ensure proper event configuration

2. **Permission denied errors**
   - Check GITHUB_TOKEN permissions
   - Verify repository settings
   - Review branch protection rules

3. **Build failures**
   - Check runner compatibility
   - Verify environment variables
   - Review dependency versions

4. **Timeout issues**
   - Increase timeout limits
   - Optimize build processes
   - Use caching strategies

### Debugging Techniques
- Enable debug logging: Set `ACTIONS_STEP_DEBUG` secret to `true`
- Use `echo` statements for variable inspection
- Check workflow run logs in GitHub UI
- Test locally with `act` tool

## 7. Interview Q&A

### Q1: What are the main components of a GitHub Actions workflow?
**A:** The main components are:
- **Workflow**: The entire automated process
- **Jobs**: Groups of steps that run on the same runner
- **Steps**: Individual tasks within a job
- **Actions**: Reusable code units
- **Runners**: Execution environments
- **Events**: Triggers that start workflows

### Q2: How do you handle secrets in GitHub Actions?
**A:** Secrets are handled through:
- Repository/Organization/Environment secrets in GitHub settings
- Accessed via `${{ secrets.SECRET_NAME }}` syntax
- Never logged or exposed in workflow outputs
- Can be scoped to specific environments for additional security

### Q3: Explain the difference between GitHub-hosted and self-hosted runners.
**A:** 
- **GitHub-hosted**: Managed by GitHub, fresh VM for each job, limited customization, included in GitHub pricing
- **Self-hosted**: Your own infrastructure, persistent environment, full customization, you manage updates and security

### Q4: How do you implement conditional execution in workflows?
**A:** Using the `if` conditional:
```yaml
- name: Deploy to production
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
  run: deploy-script.sh
```

### Q5: What is a matrix strategy and when would you use it?
**A:** Matrix strategy runs jobs across multiple configurations simultaneously. Use cases:
- Testing across multiple OS/language versions
- Building for different architectures
- Running tests with different configurations

### Q6: How do you share data between jobs?
**A:** Through artifacts:
```yaml
# Upload in one job
- uses: actions/upload-artifact@v4
  with:
    name: build-output
    path: dist/

# Download in another job
- uses: actions/download-artifact@v4
  with:
    name: build-output
```

### Q7: What are composite actions and when should you create them?
**A:** Composite actions bundle multiple steps into a reusable action. Create them when:
- You have repeated step sequences across workflows
- You want to standardize processes across repositories
- You need to share complex logic with your team

### Q8: How do you optimize workflow performance?
**A:** 
- Use caching for dependencies
- Run jobs in parallel when possible
- Use conditional execution to skip unnecessary steps
- Choose appropriate runner types
- Minimize artifact sizes

### Q9: Explain workflow triggers and provide examples.
**A:** Triggers define when workflows run:
- **push**: Code pushed to repository
- **pull_request**: PR opened/updated
- **schedule**: Cron-based timing
- **workflow_dispatch**: Manual trigger
- **release**: Release created

### Q10: How do you handle environment-specific deployments?
**A:** Using environments:
```yaml
jobs:
  deploy:
    environment: production
    steps:
      - name: Deploy
        run: deploy.sh
```
Environments can have protection rules, secrets, and approval requirements.
## 8. Comparison with Alternatives

| Feature | GitHub Actions | Jenkins | GitLab CI | Azure DevOps |
|---------|---------------|---------|-----------|--------------|
| **Hosting** | Cloud/Self-hosted | Self-hosted | Cloud/Self-hosted | Cloud/Self-hosted |
| **Configuration** | YAML | Groovy/UI | YAML | YAML/UI |
| **Integration** | Native GitHub | Plugin-based | Native GitLab | Multi-platform |
| **Cost** | Free tier generous | Open source | Free tier limited | Free tier available |
| **Learning Curve** | Low | High | Medium | Medium |
| **Marketplace** | Extensive | Large plugin ecosystem | Limited | Growing |

**When to choose GitHub Actions:**
- GitHub-centric workflow
- Simple to medium complexity projects
- Want native integration
- Prefer YAML configuration
- Need quick setup

## 9. Real-World Use Cases

### E-commerce Platform
- **Trigger**: Push to main branch
- **Process**: Run tests → Build Docker image → Deploy to staging → Run integration tests → Deploy to production
- **Features**: Blue-green deployment, rollback capabilities, Slack notifications

### Open Source Library
- **Trigger**: PR creation, push to main
- **Process**: Multi-platform testing → Security scanning → Documentation generation → NPM publishing
- **Features**: Matrix testing across Node.js versions, automated changelog generation

### Microservices Architecture
- **Trigger**: Changes in specific service directories
- **Process**: Service-specific builds → Container registry push → Kubernetes deployment → Health checks
- **Features**: Path-based triggering, service mesh integration, monitoring setup

### Infrastructure as Code
- **Trigger**: Changes to Terraform files
- **Process**: Terraform plan → Manual approval → Terraform apply → Infrastructure testing
- **Features**: Environment protection, state management, cost estimation

## 10. Study Checklist

### Must Master - Fundamentals
- [ ] YAML syntax and structure
- [ ] Workflow, job, and step concepts
- [ ] Event triggers and their use cases
- [ ] Basic actions (checkout, setup-node, etc.)
- [ ] Environment variables and secrets management

### Must Master - Intermediate
- [ ] Job dependencies and conditional execution
- [ ] Matrix strategies for multi-configuration testing
- [ ] Artifact management between jobs
- [ ] Caching strategies for performance
- [ ] Custom actions creation (composite/JavaScript/Docker)

### Must Master - Advanced
- [ ] Self-hosted runners setup and management
- [ ] Security best practices and permissions
- [ ] Reusable workflows across repositories
- [ ] Environment protection and approval gates
- [ ] Integration with cloud providers (AWS, Azure, GCP)

### Must Master - Production Ready
- [ ] Monitoring and alerting setup
- [ ] Troubleshooting common issues
- [ ] Performance optimization techniques
- [ ] Compliance and audit considerations
- [ ] Disaster recovery and rollback strategies

### Hands-on Practice
- [ ] Create a complete CI/CD pipeline for a sample project
- [ ] Implement multi-environment deployment workflow
- [ ] Build and publish a custom action
- [ ] Set up monitoring and notifications
- [ ] Practice troubleshooting failed workflows

### Interview Preparation
- [ ] Explain GitHub Actions architecture clearly
- [ ] Demonstrate workflow creation from scratch
- [ ] Compare with other CI/CD tools
- [ ] Discuss real-world implementation challenges
- [ ] Show understanding of security implications

---

**Final Tips for Mastery:**
1. Practice with real projects, not just tutorials
2. Contribute to open-source projects using GitHub Actions
3. Stay updated with new features and marketplace actions
4. Join GitHub community discussions and forums
5. Experiment with different deployment strategies and integrations
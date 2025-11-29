# CI/CD Setup Guide for Ubuntu Server

This guide covers setting up a complete CI/CD pipeline using GitHub Actions to deploy to Ubuntu EC2 instances on AWS across 3 environments: Development, Staging, and Production.

---

## Part 1: Generate Public Key from PEM File

### Step 1: Locate Your PEM File

Ensure you have your existing `.pem` file (e.g., `your-key.pem`) downloaded from AWS.

### Step 2: Generate Public Key

Open your terminal and run:

```bash
ssh-keygen -y -f /path/to/your-key.pem > your-key.pub
```

Replace `/path/to/your-key.pem` with the actual path to your PEM file.

### Step 3: Verify Public Key

View the generated public key:

```bash
cat your-key.pub
```

You should see output starting with `ssh-rsa` or `ssh-ed25519`.

---

## Part 2: Add Keys to Ubuntu EC2 Server

### Step 1: Connect to Your Server

```bash
ssh -i your-key.pem ubuntu@your-server-ip
```

### Step 2: Add Public Key to Authorized Keys

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
```

Paste your public key content, save and exit (Ctrl+X, then Y, then Enter).

### Step 3: Set Correct Permissions

```bash
chmod 600 ~/.ssh/authorized_keys
```

### Step 4: Store PEM File Securely

Keep your `.pem` file in a secure location on your local machine. You will need its content for GitHub secrets later.

---

## Part 3: GitHub Actions Workflow Configuration

### Step 1: Create Workflow Directory

In your repository root, create:

```bash
mkdir -p .github/workflows
```

### Step 2: Create Workflow File

Create `.github/workflows/deploy.yml` with support for dev, staging, and production branches.

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - dev
      - staging
      - main

jobs:
  lint-and-format:
    name: Lint and Format Check
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '22'
          cache: 'npm'

      - name: Install dependencies
        run: npm install

      - name: Install Husky
        run: npx husky install

      - name: Run Husky pre-commit hooks
        run: npx husky run pre-commit

      - name: Run linter
        run: npm run lint

      - name: Check formatting
        run: npm run format

  build:
    name: Build Application
    runs-on: ubuntu-latest
    needs: lint-and-format

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_USER: mysql_user
          MYSQL_PASSWORD: mysql_password
          MYSQL_DATABASE: test_db
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '22'
          cache: 'npm'

      - name: Install dependencies
        run: npm install
        env:
          npm_config_ignore_scripts: true

      - name: Generate Prisma client
        run: npx prisma generate
        env:
          DATABASE_URL: mysql://mysql_user:mysql_password@localhost:3306/test_db

      - name: Deploy migrations
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: mysql://mysql_user:mysql_password@localhost:3306/test_db

      - name: Build application
        run: npm run build

  deploy-dev:
    name: Deploy to Development
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/dev'
    environment: development

    steps:
      - name: Deploy to Dev Server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.DEV_HOST }}
          username: ${{ secrets.DEV_USERNAME }}
          key: ${{ secrets.DEV_SSH_KEY }}
          port: ${{ secrets.DEV_PORT }}
          script: |
            cd ${{ secrets.DEV_APP_DIRECTORY }}
            git pull origin dev
            npm install --omit=dev
            npx prisma generate
            npx prisma migrate deploy
            npm run build
            pm2 reload ${{ secrets.DEV_PM2_APP_NAME }}

  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/staging'
    environment: staging

    steps:
      - name: Deploy to Staging Server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USERNAME }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          port: ${{ secrets.STAGING_PORT }}
          script: |
            cd ${{ secrets.STAGING_APP_DIRECTORY }}
            git pull origin staging
            npm install --omit=dev
            npx prisma generate
            npx prisma migrate deploy
            npm run build
            pm2 reload ${{ secrets.STAGING_PM2_APP_NAME }}

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment: production

    steps:
      - name: Deploy to Production Server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.PRODUCTION_HOST }}
          username: ${{ secrets.PRODUCTION_USERNAME }}
          key: ${{ secrets.PRODUCTION_SSH_KEY }}
          port: ${{ secrets.PRODUCTION_PORT }}
          script: |
            cd ${{ secrets.PRODUCTION_APP_DIRECTORY }}
            git pull origin main
            npm install --omit=dev
            npx prisma generate
            npx prisma migrate deploy
            npm run build
            pm2 reload ${{ secrets.PRODUCTION_PM2_APP_NAME }}
```

---

## Part 4: Configure GitHub Repository Secrets

### Step 1: Access Repository Settings

1. Go to your GitHub repository
2. Click on "Settings"
3. Navigate to "Secrets and variables" > "Actions"

### Step 2: Create Environments

1. Go to "Environments" in the left sidebar
2. Click "New environment"
3. Create three environments: `development`, `staging`, and `production`

### Step 3: Add Development Environment Secrets

For the `development` environment (triggered by `dev` branch), add these secrets:

```
DEV_HOST = your-dev-server-ip
DEV_USERNAME = ubuntu
DEV_SSH_KEY = [content of your .pem file]
DEV_PORT = 22
DEV_APP_DIRECTORY = /home/ubuntu/your-app
DEV_PM2_APP_NAME = your-app-dev
```

### Step 4: Add Staging Environment Secrets

For the `staging` environment (triggered by `staging` branch), add these secrets:

```
STAGING_HOST = your-staging-server-ip
STAGING_USERNAME = ubuntu
STAGING_SSH_KEY = [content of your .pem file]
STAGING_PORT = 22
STAGING_APP_DIRECTORY = /home/ubuntu/your-app
STAGING_PM2_APP_NAME = your-app-staging
```

### Step 5: Add Production Environment Secrets

For the `production` environment (triggered by `main` branch), add these secrets:

```
PRODUCTION_HOST = your-production-server-ip
PRODUCTION_USERNAME = ubuntu
PRODUCTION_SSH_KEY = [content of your .pem file]
PRODUCTION_PORT = 22
PRODUCTION_APP_DIRECTORY = /home/ubuntu/your-app
PRODUCTION_PM2_APP_NAME = your-app-prod
```

### Step 6: Copy PEM File Content

To get the SSH key content:

```bash
cat your-key.pem
```

Copy the entire output including the BEGIN and END lines.

---

## Part 5: Add Deploy Key to GitHub

### Step 1: Access GitHub Account Settings

1. Click your profile picture (top right)
2. Go to "Settings"
3. Navigate to "SSH and GPG keys"

### Step 2: Add New SSH Key

1. Click "New SSH key"
2. Give it a title (e.g., "CI/CD Deploy Key")
3. Paste your public key content from `your-key.pub`
4. Click "Add SSH key"

---

## Part 6: Server Preparation

### Step 1: Install MySQL

```bash
sudo apt-get update
sudo apt-get install -y mysql-server
sudo systemctl start mysql
sudo systemctl enable mysql
```

### Step 2: Clone Your Repository

```bash
cd /home/ubuntu
git clone git@github.com:your-username/your-repo.git your-app
cd your-app
```

### Step 3: Initial Application Setup

```bash
npm install
npx prisma generate
npx prisma migrate deploy
npm run build
```

### Step 4: Start Application with PM2

For development:

```bash
pm2 start npm --name "your-app-dev" -- start
pm2 save
pm2 startup
```

For staging:

```bash
pm2 start npm --name "your-app-staging" -- start
pm2 save
pm2 startup
```

For production:

```bash
pm2 start npm --name "your-app-prod" -- start
pm2 save
pm2 startup
```

Follow the command output to enable PM2 on system restart.

### Step 5: Configure PM2 Ecosystem (Optional)

Create `ecosystem.config.js`:

```javascript
module.exports = {
  apps: [
    {
      name: 'your-app-dev',
      script: 'npm',
      args: 'start',
      env: {
        NODE_ENV: 'development',
      },
    },
    {
      name: 'your-app-staging',
      script: 'npm',
      args: 'start',
      env: {
        NODE_ENV: 'staging',
      },
    },
    {
      name: 'your-app-prod',
      script: 'npm',
      args: 'start',
      env: {
        NODE_ENV: 'production',
      },
    },
  ],
};
```

---

## Part 7: Testing the Pipeline

### Step 1: Test Development Deployment

```bash
git checkout dev
echo "test" >> test.txt
git add test.txt
git commit -m "test: CI/CD pipeline"
git push origin dev
```

### Step 2: Test Staging Deployment

```bash
git checkout staging
git merge dev
git push origin staging
```

### Step 3: Test Production Deployment

```bash
git checkout main
git merge staging
git push origin main
```

### Step 4: Monitor Workflow

1. Go to your GitHub repository
2. Click "Actions" tab
3. Watch the workflow execute through each stage for the corresponding environment

---

## Part 8: Troubleshooting

### SSH Connection Issues

If deployment fails with SSH errors:

1. Verify SSH key format in secrets (include BEGIN/END lines)
2. Check server firewall allows SSH on port 22
3. Ensure AWS security group allows inbound SSH from GitHub Actions IPs

### Build Failures

If build stage fails:

1. Check Node.js version compatibility
2. Verify all dependencies are in package.json
3. Review build logs in GitHub Actions

### PM2 Issues

If PM2 reload fails:

1. Verify PM2 app name matches secret for the environment
2. Check PM2 is running: `pm2 list`
3. Review PM2 logs: `pm2 logs`

### Permission Issues

If git pull fails:

1. Ensure server has SSH key to access GitHub
2. Verify application directory ownership: `sudo chown -R ubuntu:ubuntu /home/ubuntu/your-app`

### MySQL Connection Issues

If build fails with database errors:

1. Verify DATABASE_URL format: `mysql://user:password@localhost:3306/database`
2. Check MySQL service is running: `sudo systemctl status mysql`
3. Ensure migrations run successfully: `npx prisma migrate deploy`
4. Verify database credentials match in .env and migrations

---

## Part 9: Security Best Practices

### Rotate SSH Keys Regularly

Update your PEM file and public key every 90 days.

### Use Minimal Permissions

Ensure the deploy user only has access to necessary directories.

### Monitor Deployment Logs

Regularly review GitHub Actions logs for suspicious activity.

### Keep Secrets Updated

When changing server IPs or credentials, immediately update GitHub secrets for all 3 environments.

### Enable Branch Protection

1. Go to repository Settings > Branches
2. Add protection rules for main, staging, and dev branches
3. Require status checks to pass before merging

### Environment-Specific Considerations

- **Development**: Less strict requirements, allows frequent deployments
- **Staging**: Mirror production setup, requires approval before merging
- **Production**: Highest security, requires code review and status checks

### MySQL Security

- Use strong passwords for MySQL users
- Restrict MySQL access to localhost only
- Regularly backup databases
- Keep MySQL updated to latest patch version

---

## Part 10: Workflow Diagram

The CI/CD pipeline follows this sequence for all 3 environments:

1. Developer pushes code to dev, staging, or main branch
2. GitHub Actions triggers workflow
3. Lint stage runs code quality checks
4. Format stage verifies code formatting
5. Build stage spins up MySQL service
6. Build stage runs Prisma migrations with database connection
7. If build succeeds, appropriate deployment stage begins:
   - **dev branch** → Development environment
   - **staging branch** → Staging environment
   - **main branch** → Production environment
8. SSH connection established to target server
9. Code pulled from corresponding branch
10. Dependencies installed and Prisma client generated
11. Database migrations deployed
12. Application built
13. PM2 reloads application with zero downtime

---

## Part 11: Environment Promotion Workflow

Recommended promotion path:

```
Dev Branch → Dev Server → Code Review
    ↓
Staging Branch → Staging Server → Testing
    ↓
Main Branch → Production Server → Live
```

### Promoting to Staging

```bash
git checkout staging
git pull origin staging
git merge dev
git push origin staging
```

### Promoting to Production

```bash
git checkout main
git pull origin main
git merge staging
git push origin main
```

---

## Additional Resources

### Useful Commands

Check workflow status:

```bash
# List all workflows
gh workflow list

# View workflow runs
gh run list
```

Server management:

```bash
# Check PM2 status
pm2 status

# View application logs
pm2 logs your-app-dev

# Monitor server resources
htop

# Check MySQL status
sudo systemctl status mysql

# MySQL backup
mysqldump -u mysql_user -p database_name > backup.sql

# MySQL restore
mysql -u mysql_user -p database_name < backup.sql
```

### Environment Variables

Add application-specific environment variables:

- In GitHub environment secrets for sensitive values
- In server environment files for deployment-specific config
- In Makefile for build-time configuration
- DATABASE_URL in .env for MySQL connection

### Database Credentials

Never commit database credentials. Use:

- GitHub secrets for CI/CD
- Environment variables on servers (.env file)
- Secret management tools for production
- MySQL user permissions restricted to specific databases

# EC2-to-EC2 CI/CD Pipeline (Developer → GitHub → Production)
<img width="767" height="511" alt="image" src="https://github.com/user-attachments/assets/e2816404-0e3b-4202-b37d-0a2bc4097d38" />


This guide explains how to set up a complete CI/CD pipeline using **two EC2 instances**, **GitHub**, and **GitHub Actions**. Any code changes made on the *Developer EC2* will automatically deploy to the *Production EC2*.

---

#  STEP-1: Launch Two EC2 Instances (Developer + Production)

You need two separate EC2 machines:

## **Developer EC2**

Used for writing/editing code.

* OS: Amazon Linux 2 or Ubuntu
* Install:

  * Git
  * Apache or Nginx web server
* Host your project in:

  ```bash
  /var/www/html
  ```

## **Production EC2**

This is your live environment.

* Install:

  * Git
  * Apache or Nginx
* CI/CD pipeline will deploy updates here.

---

#  STEP-2: Create a GitHub Repository

1. Create a new repo named:
   **simple-static-website**
2. Push your code from Developer EC2:

   ```bash
   cd /var/www/html
   git init
   git remote add origin https://github.com/yourname/simple-static-website.git
   git add .
   git commit -m "initial commit"
   git push origin main
   ```

Now your code from Developer EC2 is stored in GitHub.

---

#  STEP-3: Configure SSH Access (GitHub Actions → Production EC2)

GitHub Actions needs SSH permission to deploy to Production EC2.

### **Generate an SSH Keypair on Production EC2**

```bash
ssh-keygen -t rsa -b 4096
```

Press **ENTER** for all prompts.

This creates:

| File         | Purpose                                                 |
| ------------ | ------------------------------------------------------- |
| `id_rsa`     | Private key → Upload to GitHub Secrets (`PROD_SSH_KEY`) |
| `id_rsa.pub` | Public key → Add to GitHub Deploy Keys                  |

### **Add Keys to GitHub**

* **Deploy Key** → Add the `id_rsa.pub` key (read-only or read-write)
* **Repository Secrets**:

  * `PROD_SSH_KEY` → paste the *private key*
  * `PROD_SERVER_IP` → Production EC2 public IP
  * `SSH_USERNAME` → `ec2-user` (or ubuntu)

---

#  STEP-4: Create the CI/CD Pipeline (GitHub Actions)

Create a file in your repo:

```
.github/workflows/deploy.yml
```

Paste this:

```yaml
name: Deploy Static Website to Production EC2

on:
  push:
    branches:
      - main   # Deploy whenever you push to main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout Code
      uses: actions/checkout@v3

    - name: Setup SSH Key
      run: |
        mkdir -p ~/.ssh
        echo "${{ secrets.PROD_SSH_KEY }}" > ~/.ssh/prod_key
        chmod 600 ~/.ssh/prod_key

    - name: Copy HTML Website to Production EC2
      run: |
        ssh -o StrictHostKeyChecking=no -i ~/.ssh/prod_key ${{ secrets.SSH_USERNAME }}@${{ secrets.PROD_SERVER_IP }} "mkdir -p /tmp/app"
        scp -o StrictHostKeyChecking=no -i ~/.ssh/prod_key *.html *.css *.js ${{ secrets.SSH_USERNAME }}@${{ secrets.PROD_SERVER_IP }}:/tmp/app || true

    - name: Deploy to /var/www/html (Amazon Linux)
      run: |
        ssh -o StrictHostKeyChecking=no -i ~/.ssh/prod_key ${{ secrets.SSH_USERNAME }}@${{ secrets.PROD_SERVER_IP }} "
          sudo rm -rf /var/www/html/* &&
          sudo cp -r /tmp/app/* /var/www/html/ &&
          sudo systemctl restart httpd &&
          echo 'Deployment Successful!'
```

---

#  STEP-5: How CI/CD Works

When you make any change on **Developer EC2**:

```bash
nano index.html
# OR
echo "<p>CI/CD Update</p>" >> index.html

git add .
git commit -m "new change"
git push origin main
```

### Then automatically:

1. GitHub Actions detects the push.
2. Code is securely copied to Production EC2.
3. Apache/Nginx restarts.
4. New changes appear instantly on live Production EC2.

---

#  CI/CD Completed

You now have a fully working Developer → GitHub → Production CI/CD pipeline using EC2!



# Jenkins CI/CD Pipeline: Local Jenkins + Private GitHub + AWS EC2

## 1. Project Overview

### Objective

Set up Jenkins on a local Ubuntu machine and configure an automated CI/CD pipeline.

Whenever a developer pushes code to the private GitHub repository, GitHub triggers Jenkins. Jenkins pulls the latest code, transfers it to an AWS EC2 instance, installs the required dependencies, and deploys the application.

### Complete CI/CD Flow

```text
Developer
   |
   | git push
   v
Private GitHub Repository
   |
   | GitHub Webhook
   v
Public Tunnel URL
   |
   v
Jenkins on Local Ubuntu
   |
   | Checkout + Build/Package
   | SSH / SCP using credentials
   v
AWS EC2 Instance
   |
   | Install dependencies
   | Restart application using PM2
   v
Application Running on Port 3000
```

## 2. Install Jenkins on the Local Ubuntu System

### Step 3.1: Update packages

Open the local Ubuntu terminal:

```bash
sudo apt update
sudo apt upgrade -y
```

### Step 3.2: Install Java

Jenkins requires Java.

```bash
sudo apt install fontconfig openjdk-21-jre -y
```

Verify Java:

```bash
java -version
```

### Step 3.3: Add the Jenkins repository

```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
```

Add the repository:

```bash
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" \
  | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
```

### Step 3.4: Install Jenkins

```bash
sudo apt update
sudo apt install jenkins -y
```

Start Jenkins and enable it on boot:

```bash
sudo systemctl enable --now jenkins
```

Check its status:

```bash
sudo systemctl status jenkins --no-pager
```

### Step 3.5: Open Jenkins

Open the browser on the same computer:

```text
http://localhost:8080
```

If the page does not load, check:

```bash
sudo systemctl status jenkins
sudo journalctl -u jenkins -n 50 --no-pager
```

### Step 3.6: Unlock Jenkins

Retrieve the initial administrator password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

1. Copy the password.
2. Paste it into the Jenkins unlock page.
3. Select **Install suggested plugins**.
4. Create the Jenkins administrator account.
5. Complete the setup.

### Step 3.7: Install required plugins

Go to:

**Manage Jenkins → Plugins → Available plugins**

Install these plugins if they are not already installed:

- Pipeline
- Git
- GitHub
- Credentials Binding
- SSH Agent (optional; useful for some SSH-based pipelines)

Restart Jenkins if prompted.

---

## 4. Create an AWS EC2 Instance

### Step 4.1: Open AWS EC2

1. Sign in to the AWS Console.
2. Open **EC2 → Instances**.
3. Click **Launch instances**.

### Step 4.2: Configure the instance

Example configuration:

| Setting | Value |
|---|---|
| Name | `jenkins-deployment-server` or another suitable name |
| AMI | Ubuntu Server LTS |
| Instance type | A size appropriate for the application |
| Key pair | Create or select an EC2 SSH key pair |
| Storage | Appropriate for the application |
| Security group | Allow SSH and application access as required |

For a small Node.js application, choose an instance size suitable for the application's memory and CPU needs.

### Step 4.3: Configure inbound security-group rules

| Port | Purpose | Recommended source |
|---|---|---|
| 22 | SSH administration and deployment | Your trusted public IP |
| 3000 | Node.js application | Only the clients who need access |

Do not expose port 22 to the entire internet unless there is a specific, justified requirement. Port 3000 is only needed publicly if users must access the application directly on that port.

### Step 4.4: Connect to EC2

From the local Ubuntu terminal, use the downloaded EC2 key:

```bash
chmod 400 ~/Downloads/ec2-key.pem
ssh -i ~/Downloads/ec2-key.pem ubuntu@13.234.74.131
```

Replace the key path and public IP with the actual values for your instance.

If the connection succeeds, EC2 is ready for initial setup.

---

## 5. Configure the EC2 Deployment Server

Run the following commands on EC2.

### Step 5.1: Update Ubuntu

```bash
sudo apt update
sudo apt upgrade -y
```

### Step 5.2: Install Git, Node.js, npm, and other required tools

```bash
sudo apt install -y git curl ca-certificates build-essential
```

Install a supported Node.js LTS release using an approved installation method. For example, NodeSource:

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
```

Verify:

```bash
node -v
npm -v
git --version
```

Use a Node.js version compatible with the application's `package.json`.

### Step 5.3: Install PM2

```bash
sudo npm install -g pm2
```

Verify:

```bash
pm2 -v
```

### Step 5.4: Create the deployment directory

```bash
sudo mkdir -p /var/www/html/node-hello
sudo chown ubuntu:ubuntu /var/www/html/node-hello
```

This is the directory where Jenkins will deploy the source code.

---

## 6. Generate SSH Keys for Jenkins-to-EC2 Authentication

SSH allows Jenkins to transfer code to EC2 and execute deployment commands without using an EC2 password.

There are two keys:

- **Private key:** Kept secret and added to Jenkins credentials.
- **Public key:** Added to EC2's `authorized_keys` file.

Never upload or share the private key in the GitHub repository.

### Step 6.1: Generate a dedicated SSH key on the local Ubuntu machine

Run this command on the local computer where Jenkins is installed:

```bash
ssh-keygen -t ed25519 -C "jenkins-ec2-deploy"
```

When prompted for a file location, enter:

```text
/home/YOUR_LOCAL_USERNAME/.ssh/jenkins_ec2_deploy
```

Replace `YOUR_LOCAL_USERNAME` with the local Linux username.

A passphrase-free key is convenient for non-interactive deployment, but it must be protected carefully and restricted to the deployment account and required permissions.

The two generated files will be:

```text
~/.ssh/jenkins_ec2_deploy
~/.ssh/jenkins_ec2_deploy.pub
```

The file without `.pub` is the private key. The `.pub` file is the public key.

Check that both exist:

```bash
ls -l ~/.ssh/jenkins_ec2_deploy*
```

### Step 6.2: Copy the public key to EC2

First, connect to EC2 using your existing EC2 key pair:

```bash
ssh -i ~/Downloads/ec2-key.pem ubuntu@13.234.74.131
```

On EC2, run:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
```

On the local machine, display the public key:

```bash
cat ~/.ssh/jenkins_ec2_deploy.pub
```

Copy the complete single-line public key and paste it into EC2's `authorized_keys` file. Keep existing authorized keys intact.

Save and exit `nano`:

- `Ctrl + O`
- Enter
- `Ctrl + X`

Then, on EC2:

```bash
chmod 600 ~/.ssh/authorized_keys
```

### Step 6.3: Test the new SSH key

From the local computer:

```bash
ssh -i ~/.ssh/jenkins_ec2_deploy ubuntu@13.234.74.131
```

If it connects without asking for the EC2 account password, the key authentication works.

If the key is stored in a different location, use the correct path.

### Important security note

The above key is generated on the local machine. Jenkins must be able to read the private key when the pipeline runs. The easiest approach is to copy its private-key contents into a Jenkins **SSH Username with private key** credential. Do not put the private key in the repository or a `Jenkinsfile`.

For stronger isolation, use a dedicated deployment account and limit its permissions instead of granting broad administrative access.

---

## 7. Generate a GitHub Personal Access Token (PAT)

Jenkins needs authentication to read a private GitHub repository.

### Step 7.1: Open GitHub token settings

1. Sign in to GitHub.
2. Open your profile menu.
3. Select **Settings**.
4. Go to **Developer settings**.
5. Select **Personal access tokens**.
6. Choose **Fine-grained tokens**.
7. Click **Generate new token**.

### Step 7.2: Configure the token

Example:

| Field | Value |
|---|---|
| Token name | `jenkins-private-repo-read` |
| Resource owner | Your GitHub account or organization |
| Repository access | Only select repositories |
| Selected repository | `node-hello-setup` |
| Expiration | Choose an appropriate expiry date |

Under **Repository permissions**, configure:

| Permission | Access |
|---|---|
| Contents | Read-only |
| Metadata | Read-only (normally enabled automatically) |

These permissions are sufficient for Jenkins to clone or fetch source code from the selected private repository.

Do not grant write access unless the pipeline actually needs to modify repository contents.

Click **Generate token** and copy the token immediately. GitHub generally will not display the full token again.

Keep the token secret. Do not put it in the `Jenkinsfile`, source code, screenshots, or Git repository.

If the repository belongs to an organization, its token approval policies may also need to be satisfied.

---

## 8. Add GitHub Credentials in Jenkins

### Step 8.1: Open Jenkins credentials

1. Open `http://localhost:8080`.
2. Go to **Manage Jenkins**.
3. Open **Credentials**.
4. Select the appropriate global or system store.
5. Open the relevant domain, usually **Global credentials (unrestricted)**.
6. Click **Add Credentials**.

### Step 8.2: Add the GitHub token

Choose:

| Field | Value |
|---|---|
| Kind | Username with password |
| Username | Your GitHub username |
| Password | The GitHub PAT |
| ID | `github-private` |
| Description | `GitHub private repository read access` |

Click **Create**.

The pipeline will use this credential ID to authenticate to GitHub. The actual token is stored in Jenkins, not in the `Jenkinsfile`.

### Why use `github-private`?

The pipeline references the credential by ID:

```groovy
git branch: 'master',
    credentialsId: 'github-private',
    url: 'https://github.com/DhruvShah0612/node-hello-setup.git'
```

The ID must match exactly.

---

## 9. Add EC2 SSH Credentials in Jenkins

### Step 9.1: Open Add Credentials

Go to:

**Manage Jenkins → Credentials → Global credentials → Add Credentials**

### Step 9.2: Configure the credential

Choose:

| Field | Value |
|---|---|
| Kind | SSH Username with private key |
| Scope | Global, or the narrowest suitable scope |
| Username | `ubuntu` |
| Private Key | Enter directly |
| Private key contents | Paste the complete private key |
| Passphrase | Leave blank if the key has no passphrase |
| ID | `ec2-ssh` |
| Description | `SSH access for EC2 deployment` |

To copy the private key contents on the local computer:

```bash
cat ~/.ssh/jenkins_ec2_deploy
```

Copy the complete output, including the `BEGIN` and `END` lines, and paste it into the Jenkins credential's private-key field.

Click **Create**.

**Important:** The `ec2-ssh` credential ID must match the ID referenced in the pipeline. Never paste the private key into the repository or console logs.

### How the two Jenkins credentials work

- `github-private`: Jenkins uses this to read the private GitHub repository.
- `ec2-ssh`: Jenkins uses this to authenticate to EC2 and transfer/deploy the application.

They serve different purposes.

---

## 10. Prepare the GitHub Repository

Example repository:

```text
https://github.com/DhruvShah0612/node-hello-setup
```

Branch:

```text
master
```

A typical Node.js repository may contain:

```text
node-hello-setup/
├── app.js
├── package.json
├── package-lock.json
├── Jenkinsfile
└── README.md
```

Your actual application entry point may be `server.js` or another file. Confirm the correct entry point before deploying.

Ensure that `package.json` contains the required start script. Example:

```json
{
  "scripts": {
    "start": "node app.js"
  }
}
```

If your project uses `server.js`, change the script accordingly.

Add a `.gitignore` file to avoid committing unnecessary or sensitive files:

```gitignore
node_modules/
.env
*.log
.DS_Store
```

Do not commit passwords, API keys, tokens, private keys, or production `.env` files.

### Push code to GitHub

From the local project directory:

```bash
git add .
git commit -m "Configure Jenkins deployment"
git push origin master
```

The `master` branch must match the branch configured in Jenkins.

---

## 11. Create the Jenkins Pipeline

### Step 11.1: Create a new pipeline job

1. Open Jenkins.
2. Click **New Item**.
3. Enter the job name:

   `node-hello-setup-pipeline`

4. Select **Pipeline**.
5. Click **OK**.

### Step 11.2: Configure the pipeline

For a simple setup:

1. Open the job's **Configure** page.
2. Find the **Pipeline** section.
3. Under Definition, choose **Pipeline script**.
4. Paste the pipeline script below.
5. Save the job.

This approach stores the pipeline script in Jenkins. For a more maintainable setup, you can later store the `Jenkinsfile` in GitHub and choose **Pipeline script from SCM**.

### Step 11.3: Example deployment pipeline

The following example assumes the repository's entry point is `app.js`, the application is configured to run on port `3000`, and PM2 is installed on EC2.

```groovy
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        REPO_URL = 'https://github.com/DhruvShah0612/node-hello-setup.git'
        BRANCH = 'master'
        EC2_HOST = '13.234.74.131'
        DEPLOY_DIR = '/var/www/html/node-hello'
        APP_NAME = 'node-hello'
        APP_PORT = '3000'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: "${BRANCH}",
                    credentialsId: 'github-private',
                    url: "${REPO_URL}"
            }
        }

        stage('Validate Source') {
            steps {
                sh '''
                    set -eu
                    test -f package.json
                    test -f app.js
                    echo "Source validation passed."
                '''
            }
        }

        stage('Package Source') {
            steps {
                sh '''
                    set -eu
                    tar czf /tmp/node-hello.tar.gz \
                        --exclude=.git \
                        --exclude=node_modules \
                        --exclude=.env \
                        --exclude='*.log' \
                        .
                '''
            }
        }

        stage('Deploy to EC2') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ec2-ssh',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {
                    sh '''
                        set -eu
                        chmod 600 "$SSH_KEY"

                        scp -i "$SSH_KEY" \
                            -o StrictHostKeyChecking=accept-new \
                            /tmp/node-hello.tar.gz \
                            "$SSH_USER@$EC2_HOST:/tmp/node-hello.tar.gz"

                        ssh -i "$SSH_KEY" \
                            -o StrictHostKeyChecking=accept-new \
                            "$SSH_USER@$EC2_HOST" \
                            "DEPLOY_DIR='$DEPLOY_DIR' APP_NAME='$APP_NAME' APP_PORT='$APP_PORT' bash -s" <<'REMOTE'
set -eu

mkdir -p "$DEPLOY_DIR"
tar xzf /tmp/node-hello.tar.gz -C "$DEPLOY_DIR"
rm -f /tmp/node-hello.tar.gz

cd "$DEPLOY_DIR"

npm install --omit=dev

if pm2 describe "$APP_NAME" >/dev/null 2>&1; then
    pm2 delete "$APP_NAME"
fi

export PORT="$APP_PORT"

pm2 start app.js \
    --name "$APP_NAME" \
    --update-env

pm2 save

sleep 3

curl --connect-timeout 3 --max-time 10 \
    -fsS "http://127.0.0.1:$APP_PORT/" >/dev/null

echo "Deployment and local HTTP health check succeeded."
REMOTE
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Deployment completed successfully.'
            echo 'Application URL: http://13.234.74.131:3000/'
        }

        failure {
            echo 'Deployment failed. Review the Jenkins Console Output.'
        }

        always {
            sh 'rm -f /tmp/node-hello.tar.gz'
        }
    }
}
```

## 12. Configure GitHub Webhook for Automatic Triggering

The goal is to start Jenkins automatically when a push occurs on GitHub.

### Step 12.1: Enable webhook triggering in Jenkins

Open the Jenkins pipeline job:

1. Click **Configure**.
2. Find **Build Triggers**.
3. Select **GitHub hook trigger for GITScm polling**.
4. Save.

### Step 12.2: Make local Jenkins reachable from GitHub

GitHub cannot send a webhook to:

```text
http://localhost:8080
```

That address only works on the local computer.

For a basic demonstration, you can use LocalTunnel to expose the local Jenkins port.

Install/use Node.js and npm on the local computer, then run:

```bash
npx localtunnel --port 8080 --local-host 127.0.0.1
```

Keep this terminal running.

LocalTunnel will display a public HTTPS URL, for example:

```text
https://example-name.loca.lt
```

Use the actual URL displayed by your terminal. Do not copy the example URL.

### Step 12.3: Add the webhook in GitHub

1. Open the private repository on GitHub.
2. Go to **Settings**.
3. Open **Webhooks**.
4. Click **Add webhook**.

Configure:

| Field | Value |
|---|---|
| Payload URL | `https://YOUR-PUBLIC-TUNNEL-URL/github-webhook/` |
| Content type | `application/json` |
| Secret | Optional shared webhook secret, if configured and supported by the Jenkins integration |
| Events | Just the push event |
| Active | Enabled |

Replace `YOUR-PUBLIC-TUNNEL-URL` with the actual tunnel hostname.

For example, if the tunnel URL is `https://example-name.loca.lt`, the payload URL becomes:

```text
https://example-name.loca.lt/github-webhook/
```

Click **Add webhook**.

The GitHub PAT is not the webhook secret. The PAT authenticates Jenkins when reading the repository; the webhook delivers the push notification.

### Step 12.4: Test the webhook

1. Open GitHub **Settings → Webhooks**.
2. Select the webhook.
3. Review its recent deliveries.
4. Check the response and delivery status.
5. Push a new commit to the configured branch.

If the webhook reaches Jenkins and the job is configured correctly, the pipeline should start automatically.

If LocalTunnel cannot connect or returns a connection error, first verify Jenkins locally:

```bash
curl -I http://127.0.0.1:8080
sudo systemctl status jenkins --no-pager
```

---

## 13. End-to-End Testing

Once the setup is complete, test the entire flow.

### Step 13.1: Change the code

Make a small change to the application, such as a homepage message.

### Step 13.2: Push the change

```bash
git add .
git commit -m "Test automatic deployment"
git push origin master
```

### Step 13.3: Verify the Jenkins build

Open:

```text
http://localhost:8080
```

Open `node-hello-setup-pipeline` and check the build history.

Review **Console Output** for:

- Successful Git checkout
- Source validation
- Packaging
- SSH/SCP transfer
- Dependency installation
- PM2 restart
- HTTP health check

### Step 13.4: Verify the application on EC2

Connect to EC2:

```bash
ssh -i ~/.ssh/jenkins_ec2_deploy ubuntu@13.234.74.131
```

Check PM2:

```bash
pm2 status
pm2 logs node-hello --lines 50
```

Check the application locally on EC2:

```bash
curl -i http://127.0.0.1:3000/
```

If the security group allows public application access, test in a browser:

```text
http://13.234.74.131:3000/
```

For production use, prefer HTTPS through a reverse proxy instead of exposing the application directly on port 3000.

---

## 14. Final Architecture Summary

| Component | Responsibility |
|---|---|
| Local Ubuntu | Runs Jenkins |
| Jenkins | Executes the CI/CD pipeline |
| GitHub private repository | Stores application code |
| `github-private` | Authenticates Jenkins to GitHub |
| `ec2-ssh` | Authenticates Jenkins to EC2 |
| GitHub webhook | Notifies Jenkins about a push |
| LocalTunnel | Provides a temporary public route to local Jenkins |
| AWS EC2 | Hosts the deployed application |
| PM2 | Runs and manages the Node.js process |
| Port 3000 | Serves the example application |

### Final Result

After setup, the intended workflow is:

1. Developer pushes code to the configured GitHub branch.
2. GitHub sends a webhook notification to Jenkins.
3. Jenkins starts the pipeline.
4. Jenkins authenticates to GitHub and fetches the latest code.
5. Jenkins packages the source and transfers it to EC2 using SSH/SCP.
6. EC2 installs the required dependencies and restarts the PM2 application.
7. Jenkins checks whether the application responds successfully.

**Expected application URL for this example:**

`http://13.234.74.131:3000/`

The automatic trigger depends on GitHub being able to reach the webhook URL. The deployment succeeds only when the repository, credentials, SSH access, application configuration, and EC2 networking are correctly configured.

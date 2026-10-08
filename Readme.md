# Node.js CI/CD Pipeline with Jenkins

Jenkins pipeline that clones a simple Express app from GitHub, copies it to a target server over SSH, installs dependencies, and runs it with PM2.

## Pipeline Flow

```
Git Push → Jenkins → Clone Repo → Upload (SCP) → npm install → PM2 start/restart
```

## Architecture

```
GitHub ──► Jenkins Server ──SSH/SCP──► Target Server (EC2)
                                         └─ Node.js + PM2 → app on :3000
```

## Project Structure

```
.
├── app.js          # Express app
├── package.json
├── Jenkinsfile     # Pipeline definition
└── README.md
```

## Prerequisites

**Jenkins server**
- Plugins: **Git**, **Pipeline**, **SSH Agent**
- SSH private key stored as a credential (ID: `node-pipeline-key`)

**Target server**
- Ubuntu with Node.js, npm, and PM2 installed
- Jenkins public key added to `~/.ssh/authorized_keys` of user `ubuntu`
- Security group: allow **port 22** from Jenkins, **port 3000** from users

Install on target:
```bash
sudo apt update && sudo apt install -y nodejs npm
sudo npm install -g pm2
```

## App Code

**app.js**
```js
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
  res.send('Hiii from jenkins, This is my first pipeline');
});

app.listen(port, () => {
  console.log(`App listening at http://localhost:${port}`);
});
```

**package.json**
```json
{
  "name": "jenkins-node-app",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

## Jenkinsfile

```groovy
pipeline {
    agent any

    environment {
        SERVER_IP      = '172.31.47.151'
        SSH_CREDENTIAL = 'node-pipeline-key'
        REPO_URL       = 'https://github.com/Ritesh123-a/jenkin-pipeline--nodejs'
        BRANCH         = 'main'
        REMOTE_USER    = 'ubuntu'
        REMOTE_PATH    = '/home/ubuntu/node-app'
    }

    stages {
        stage('Clone Repository') {
            steps {
                git branch: "${BRANCH}", url: "${REPO_URL}"
            }
        }

        stage('Upload Files to target-server') {
            steps {
                sshagent([SSH_CREDENTIAL]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${SERVER_IP} 'mkdir -p ${REMOTE_PATH}'
                        scp -o StrictHostKeyChecking=no -r * ${REMOTE_USER}@${SERVER_IP}:${REMOTE_PATH}/
                    """
                }
            }
        }

        stage('Install Dependencies & Start App on the target server') {
            steps {
                sshagent([SSH_CREDENTIAL]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${SERVER_IP} '
                            cd ${REMOTE_PATH} &&
                            npm install &&
                            pm2 start app.js --name node-app || pm2 restart node-app
                        '
                    """
                }
            }
        }
    }

    post {
        success { echo '✅ Application deployed successfully!' }
        failure { echo '❌ Deployment failed.' }
    }
}
```

## Environment Variables

| Variable | Purpose |
|---|---|
| `SERVER_IP` | Target server IP (private IP if same VPC) |
| `SSH_CREDENTIAL` | Jenkins credential ID of SSH key |
| `REPO_URL` | GitHub repo to clone |
| `BRANCH` | Branch to deploy |
| `REMOTE_USER` | SSH user on target |
| `REMOTE_PATH` | App folder on target |

## Jenkins Setup

1. **Install plugin:** Manage Jenkins → Plugins → **SSH Agent**.
2. **Add credential:** Manage Jenkins → Credentials → Global → Add
   - Kind: *SSH Username with private key*
   - ID: `node-pipeline-key`
   - Username: `ubuntu` → paste private key → Save.
3. **Create job:** New Item → name → **Pipeline** → OK.
4. **Link repo:** Pipeline → *Pipeline script from SCM* → Git → repo URL → Branch `*/main` → Script Path `Jenkinsfile`.
5. **Save** → **Build Now**.

## Auto-Trigger on Push (Optional)

**GitHub webhook**
- Repo → Settings → Webhooks → Add
- Payload URL: `http://<JENKINS_URL>/github-webhook/`
- Content type: `application/json` → Event: *Push*

**Jenkins job**
- Build Triggers → **GitHub hook trigger for GITScm polling**

## Verify Deployment

```bash
# on target server
pm2 list
pm2 logs node-app

# from browser
http://<TARGET_PUBLIC_IP>:3000
```

Expected output: `Hiii from jenkins, This is my first pipeline`

## Troubleshooting

| Problem | Fix |
|---|---|
| `Permission denied (publickey)` | Add Jenkins public key to target `authorized_keys`; check credential ID |
| `pm2: command not found` | Install PM2 globally on target; use `sudo npm i -g pm2` |
| `npm: command not found` | Install Node.js on target |
| `Host key verification failed` | Keep `-o StrictHostKeyChecking=no` flag |
| `Connection timed out` | Open port 22 in security group; check `SERVER_IP` |
| App not opening in browser | Open port 3000 in security group; check `pm2 logs` |
| `sshagent` not found | Install **SSH Agent** plugin |

## Known Limits & Improvements

- `scp -r *` skips hidden files and copies the Jenkinsfile too; use `rsync -av --exclude node_modules` instead
- `StrictHostKeyChecking=no` is fine for learning; use `known_hosts` in production
- Add `npm test` stage before deploy
- Run `pm2 save` and `pm2 startup` so the app survives server reboot
- Add Slack/email notifications in `post`

## License

MIT
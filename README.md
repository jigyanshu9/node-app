# node-app

A full-stack Node.js application (React + Vite frontend, Node.js backend) packaged with Docker and deployed automatically through a Jenkins CI/CD pipeline running on AWS EC2.

Every commit pushed to GitHub triggers Jenkins, which builds the Docker image, runs checks, pushes the image to Docker Hub and redeploys the running container.

## Tech Stack

| Area | Tools |
|---|---|
| Frontend | React, Vite |
| Backend | Node.js (`server.js`) serving the built frontend |
| Containerization | Docker |
| CI/CD | Jenkins (Declarative Pipeline) |
| Registry | Docker Hub |
| Hosting | AWS EC2 (Ubuntu) |
| Source control | Git, GitHub (webhook trigger) |

## Project Structure

```
node-app/
├── backend/          # Node.js server (server.js, package.json)
├── frontend/         # React + Vite source
├── Dockerfile        # Builds frontend, bundles it into the backend image
├── .dockerignore
├── jenkinsfile       # Jenkins pipeline definition
└── README.md
```

## How the Docker Image Is Built

1. Start from `node:20-alpine`.
2. Install frontend dependencies and run `npm run build` (Vite).
3. Install backend production dependencies (`npm ci --omit=dev`).
4. Copy the built frontend (`frontend/dist`) into `backend/public`.
5. Run the server as a non-root user on port **3000**.

## Run Locally

### With Docker

```bash
docker build -t node-app .
docker run -d --name node-app -p 3000:3000 node-app
```

Open http://localhost:3000.

### Without Docker

```bash
# Frontend
cd frontend
npm install
npm run build
cp -r dist ../backend/public

# Backend
cd ../backend
npm install
npm start
```

## CI/CD Pipeline

The pipeline is defined in `jenkinsfile` and runs these stages:

| Stage | What it does |
|---|---|
| Checkout | Pulls the latest code from GitHub |
| Build | Builds the Docker image, tagged with the build number and `latest` |
| Test | Checks `server.js` for syntax errors and confirms the frontend build exists in `public/` |
| Push Image | Logs in to Docker Hub and pushes both tags |
| Deploy | Removes the old container and starts the new one on port 3000 |
| Smoke Check | Requests the app with `curl` to confirm it is responding |

### Triggers

- **GitHub webhook** (`githubPush()`): starts a build instantly on every push.
- **SCM polling** (`pollSCM('H/2 * * * *')`): backup that checks for new commits about every 2 minutes if the webhook fails.

## Jenkins Server Setup (EC2)

1. Launch an Ubuntu EC2 instance (`t3.small` or larger, 20 GB disk).
2. Open inbound ports in the security group: **22** (SSH, your IP only), **8080** (Jenkins) and **3000** (app).
3. Install Java 21, Docker and Jenkins (use the current Jenkins apt repository key from jenkins.io).
4. Add the Jenkins user to the Docker group and restart Jenkins:

   ```bash
   sudo usermod -aG docker jenkins
   sudo systemctl restart jenkins
   ```

5. Open `http://<ec2-public-ip>:8080`, unlock Jenkins and install the suggested plugins.

## Jenkins Configuration

1. **Credentials:** add a *Username with password* credential with ID `dockerhub-creds` (Docker Hub username and an access token with Read & Write permission).
2. **Job:** create a Pipeline job named `myapp-pipeline`:
   - Definition: *Pipeline script from SCM*
   - SCM: Git, repository `https://github.com/jigyanshu9/node-app.git`
   - Branch: `*/master`
   - Script Path: `jenkinsfile`
3. **Webhook:** in GitHub go to *Settings → Webhooks → Add webhook*:
   - Payload URL: `http://<ec2-public-ip>:8080/github-webhook/`
   - Content type: `application/json`
   - Event: push
4. Run **Build Now** once so Jenkins registers the triggers from the Jenkinsfile.

In the `jenkinsfile`, set `DOCKERHUB_USER` to your Docker Hub username.

## Verify a Deployment

```bash
docker ps
curl http://localhost:3000
```

Then open `http://<ec2-public-ip>:3000` in a browser and check that the new tag appears at `hub.docker.com/r/<your-dockerhub-username>/node-app`.

## Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| `Unable to find Jenkinsfile` | File name or Script Path mismatch (the name is case-sensitive), or wrong branch |
| `unexpected char: '\'` in the Jenkinsfile | Markdown code fences (triple backticks) were copied into the file; remove them |
| `Could not find credentials entry with ID 'dockerhub-creds'` | The credential is missing or its ID is spelled differently |
| `permission denied ... docker.sock` | Jenkins user is not in the `docker` group; add it and restart Jenkins |
| Push `denied` / `unauthorized` | Wrong `DOCKERHUB_USER`, or the token lacks write permission |
| Webhook shows a red cross | EC2 public IP changed (use an Elastic IP) or port 8080 is closed in the security group |
| Build does not start on commit | Webhook missing, or no manual build has been run since editing the Jenkinsfile |

## Author

Jigyanshu Pradhan

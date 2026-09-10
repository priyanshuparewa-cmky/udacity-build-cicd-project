# Movie Picture Pipeline - Complete Step-by-Step Guide

This guide corresponds directly to the 14 steps outlined in the Udacity Project Specification.

---

## 1. Project Workspace Overview
The workspace structure contains:
- `starter/frontend`: React TypeScript frontend application with Dockerfile, ESLint configuration, Jest tests, and Kubernetes manifests (`k8s/`).
- `starter/backend`: Python Flask backend application with Dockerfile, Flake8 configuration, Pytest test suite, and Kubernetes manifests (`k8s/`).
- `setup/terraform`: Terraform configuration creating VPC, ECR repositories (`frontend` and `backend`), EKS cluster (`cluster`), node group, and IAM user (`github-action-user`).
- `setup/init.sh`: Script to grant the `github-action-user` IAM user access to EKS (`aws-auth` ConfigMap).
- `.github/workflows/`:
  - `frontend-ci.yaml`: Triggered on PRs to `main` modifying `starter/frontend/**`.
  - `backend-ci.yaml`: Triggered on PRs to `main` modifying `starter/backend/**`.
  - `backend-cd.yaml`: Triggered on merge/push to `main` modifying `starter/backend/**`.
  - `frontend-cd.yaml`: Triggered on merge/push to `main` modifying `starter/frontend/**`.

---

## 2. Connect to GitHub and Push Starter Project
1. In your workspace terminal (or Udacity VS Code terminal):
   ```bash
   git init
   git config --global user.email "your-email@example.com"
   git config --global user.name "Your Name"
   git add .
   git commit -m "feat: complete CI/CD workflows and project setup"
   ```
2. Create a **public** repository on GitHub (required for free GitHub Actions minutes):
   - Using GitHub CLI in workspace:
     ```bash
     gh auth login
     gh repo create udacity-build-cicd-project --public --source=. --remote=origin --push
     ```
   - Or create a repo via https://github.com/new and push:
     ```bash
     git remote add origin https://github.com/<YOUR_USERNAME>/<YOUR_REPO_NAME>.git
     git branch -M main
     git push -u origin main
     ```

---

## 3. Test Frontend CI
1. Create a feature branch:
   ```bash
   git checkout -b test-frontend-ci
   ```
2. Make a minor change or test edit in `starter/frontend/src/App.js` or `README.md`.
3. Commit and push:
   ```bash
   git commit -am "test: trigger frontend CI"
   git push origin test-frontend-ci
   ```
4. Open a Pull Request from `test-frontend-ci` into `main` on GitHub.
5. In GitHub Actions, observe:
   - Parallel execution of `Lint Frontend` and `Test Frontend`.
   - `Build Frontend` running only after both pass, building the Docker image with `--build-arg REACT_APP_MOVIE_API_URL=http://localhost:5000`.
6. Once green, merge the pull request.

---

## 4. Test Backend CI
1. Create a feature branch:
   ```bash
   git checkout -b test-backend-ci
   ```
2. Make a minor change or test edit in `starter/backend/movies/movies_api.py`.
3. Commit and push:
   ```bash
   git commit -am "test: trigger backend CI"
   git push origin test-backend-ci
   ```
4. Open a Pull Request from `test-backend-ci` into `main` on GitHub.
5. In GitHub Actions, observe:
   - Parallel execution of `Lint Backend` and `Test Backend`.
   - `Build Backend` running only after both pass, building `mp-backend:latest`.
6. Once green, merge the pull request.

---

## 5 & 6. Prepare Terraform & Create AWS Infrastructure
1. In the Udacity workspace terminal, configure your temporary AWS credentials provided by the Cloud Gateway.
2. Initialize and apply Terraform:
   ```bash
   cd setup/terraform
   terraform init
   terraform validate
   terraform plan
   terraform apply -auto-approve
   ```
3. Inspect outputs:
   ```bash
   terraform output
   ```
4. Verify EKS worker node is Ready:
   ```bash
   aws eks update-kubeconfig --name cluster --region us-east-1
   kubectl get nodes
   ```
   *(Wait until the worker node status is `Ready`)*

---

## 7. Connect GitHub Actions to AWS/EKS
1. In the AWS Console (Cloud Gateway), go to the **IAM** service.
2. Click **Users** -> select **`github-action-user`**.
3. Navigate to **Security Credentials** tab -> **Access keys** -> **Create access key**.
4. Select **Application running outside AWS**, click **Next**, and click **Create access key**.
5. Go to your GitHub repository -> **Settings** -> **Secrets and variables** -> **Actions** -> **New repository secret**:
   - `AWS_ACCESS_KEY_ID`: `<Access Key ID>`
   - `AWS_SECRET_ACCESS_KEY`: `<Secret Access Key>`
6. Back in your Udacity workspace terminal, run `setup/init.sh` to grant the user permissions in Kubernetes:
   ```bash
   cd setup
   chmod +x init.sh
   ./init.sh
   ```
   *(You should see `Done!`)*

---

## 8. Run Backend CD & Verify Deployment
1. Either manually trigger `Backend CD` via the **Actions** tab on GitHub (`Run workflow`), or push a commit to `main` impacting `starter/backend/**`.
2. Wait for `Backend CD` to complete successfully:
   - Lint & Test pass.
   - Docker image built, tagged with commit SHA, pushed to ECR `backend`.
   - Manifests updated via Kustomize and applied to EKS.
3. Verify the Backend LoadBalancer in terminal:
   ```bash
   kubectl get svc backend
   ```
4. Copy the external hostname and test `/movies`:
   ```bash
   curl http://<BACKEND_LOADBALANCER_HOSTNAME>/movies
   ```
   Expected response:
   ```json
   {"movies":[{"id":"123","title":"Top Gun: Maverick"},{"id":"456","title":"Sonic the Hedgehog"},{"id":"789","title":"A Quiet Place"}]}
   ```
   *(Note: The root `/` endpoint may return 404 by design; verify `/movies`)*

---

## 9 & 10. Run Frontend CD & Verify UI
1. (Optional) Set the GitHub repository secret `REACT_APP_MOVIE_API_URL` to `http://<BACKEND_LOADBALANCER_HOSTNAME>` (the workflow also automatically discovers the backend LoadBalancer hostname from EKS if not set).
2. Either manually trigger `Frontend CD` via the **Actions** tab on GitHub, or push a commit to `main` impacting `starter/frontend/**`.
3. Wait for `Frontend CD` to turn green.
4. Get the Frontend LoadBalancer service:
   ```bash
   kubectl get svc frontend
   ```
5. Open `http://<FRONTEND_LOADBALANCER_HOSTNAME>` in your browser.
6. Verify the UI loads and displays:
   - **Top Gun: Maverick**
   - **Sonic the Hedgehog**
   - **A Quiet Place**

---

## 11 & 12. Evidence Checklist for Udacity Submission
Take clear screenshots of the following:
1. [ ] **Frontend CI** green workflow run on GitHub Actions.
2. [ ] **Backend CI** green workflow run on GitHub Actions.
3. [ ] **Backend CD** green workflow run on GitHub Actions.
4. [ ] **Frontend CD** green workflow run on GitHub Actions.
5. [ ] **Backend `/movies` response** (e.g. `curl http://<BACKEND_URL>/movies` in terminal or browser showing JSON).
6. [ ] **Frontend working page** showing the movie list in browser.
7. [ ] **Kubernetes pods and services** running (`kubectl get pods,svc -o wide`).
8. [ ] **GitHub repository link** (public).

---

## 13. Destroy AWS Resources
Once all required evidence and screenshots are captured, destroy AWS resources to avoid consuming cloud lab credits:
```bash
cd setup/terraform
terraform destroy -auto-approve
```

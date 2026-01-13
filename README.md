# MLOPs_Project

A compact end-to-end MLOps example: data (DVC) → training → MLflow tracking & registry → Docker images → GitHub Actions CI/CD → Kubernetes deployment.

## Contents
- `api/` — FastAPI service
- `ui/` — Streamlit web UI
- `k8s/` — Kubernetes manifests for API and UI
- `docker/` — Dockerfiles for API and UI
- `.github/workflows/` — GitHub Actions CI/CD pipeline (`mlops-830cicd.yaml`)
- `diagrams/` — draw.io XML diagrams for architecture and flows

## Quickstart (local/minimal)
Prerequisites: Python 3.9+, Docker, kubectl (optional), DVC (optional), Git.

1. Clone the repo:
	git clone <repo-url>
2. Create and activate a Python venv and install requirements (if any):
	python -m venv .venv
	source .venv/bin/activate
	pip install -r requirements.txt  # if present

3. Run the API locally:
	cd api
	uvicorn main:app --reload --host 0.0.0.0 --port 8000

4. Run the UI locally (Streamlit):
	cd ui
	python -m streamlit run app.py --server.address 0.0.0.0

Note: If `streamlit` command is not found, install it in the active environment: `python -m pip install streamlit`.

## DVC & Data
- Raw data is tracked via DVC (`data/raw/*.dvc`). Use `dvc pull` / `dvc push` to sync remote storage.
- To add data locally: `dvc add data/raw/<file>` then `dvc push` (requires remote configured in `dvc.yaml` / `.dvc/config`).

## MLflow
- Training scripts log to MLflow (see `src/train.py`). Configure MLflow tracking URI and registry as needed.

## CI/CD (GitHub Actions)
- The pipeline `mlops-830cicd.yaml` builds Docker images for API and UI and pushes them to DockerHub (credentials via `secrets.DOCKER_USERNAME` and `secrets.DOCKER_PASSWORD`).
- The `deploy` job runs on a self-hosted runner and applies Kubernetes manifests.

Secrets required for deployment (set in GitHub repo Settings → Secrets → Actions):
- `DOCKER_USERNAME`, `DOCKER_PASSWORD` — for DockerHub push
- `KUBECONFIG_DATA` — a single-line base64 of `/etc/kubernetes/admin.conf` (preferred) or the raw kubeconfig YAML; workflow decodes both forms.

To generate the preferred secret on the cluster master:
```bash
base64 -w0 /etc/kubernetes/admin.conf > kube.b64
# copy the single-line content and save it as the GitHub secret KUBECONFIG_DATA
```

## Self-hosted GitHub Actions runner
1. On the machine that will host the runner (recommended: non-root user such as `devopsadmin`): download and extract the GitHub Actions runner into `/home/devopsadmin/actions-runner`.
2. Generate a registration token in the repo Settings → Actions → Runners → New self-hosted runner and run `./config.sh --url <repo-url> --token <token>` as the runner user.
3. Test with `./run.sh` and install as a service with `sudo ./svc.sh install && sudo ./svc.sh start`.

## Kubernetes access for runner user
On the master node (run as root) install a per-user kubeconfig so the runner can run `kubectl`:
```bash
mkdir -p /home/devopsadmin/.kube
cp /etc/kubernetes/admin.conf /home/devopsadmin/.kube/config
chown devopsadmin:devopsadmin /home/devopsadmin/.kube/config
chmod 600 /home/devopsadmin/.kube/config
```
Or create a limited service-account and kubeconfig for least-privilege access (recommended for production).

## Troubleshooting
- `streamlit: command not found` — ensure Streamlit is installed in the active virtualenv or run via `python -m streamlit`.
- `kubectl` errors about invalid kubeconfig — ensure `KUBECONFIG_DATA` is correctly encoded or that `/home/<runner>/.kube/config` contains a valid kubeconfig YAML (not a raw base64 blob).
- `sudo: password for <user>` failures — ensure the user is in the `sudo` group or use cloud provider recovery to edit sudoers if locked out.

## Diagrams
- See `diagrams/` for draw.io XML files you can import into diagrams.net (draw.io). Files:
  - `mlops_architecture.xml`
  - `dataflow_dvc_s3_mlflow.xml`
  - `github_actions_flow.xml`
  - `ui_api_k8s_mlflow_s3.xml`
  - `fastapi_prometheus_grafana.xml`

## Next steps & recommendations
- Replace admin kubeconfig with a scoped service account kubeconfig for CI/CD deployments.
- Add `requirements.txt` and small README sections per component (api, ui) if you want runnable examples for newcomers.



### Docker Repo ##

https://hub.docker.com/repository/docker/avinashacharya26/loksai-mlops830ui/general
https://hub.docker.com/repository/docker/avinashacharya26/loksai-mlops830api/general


#### kubectl outputs ####

root@kmaster-node:~/kubernetes/actions-runner# kubectl get pods
NAME                                 READY   STATUS             RESTARTS      AGE
loksai-api-deploy-7dcc7b7b7c-gzfqx   0/1     CrashLoopBackOff   6 (36s ago)   6m57s
loksai-ui-deploy-59bfff8f6f-zzqp2    1/1     Running            0             6m57s
root@kmaster-node:~/kubernetes/actions-runner# 
root@kmaster-node:~/kubernetes/actions-runner# 
root@kmaster-node:~/kubernetes/actions-runner# 
root@kmaster-node:~/kubernetes/actions-runner# kubectl get deploy
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
loksai-api-deploy   0/1     1            0           9m24s
loksai-ui-deploy    1/1     1            1           9m23s
root@kmaster-node:~/kubernetes/actions-runner# kubectl get svc
NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes       ClusterIP   10.96.0.1        <none>        443/TCP          5h14m
loksai-api-svc   ClusterIP   10.102.133.73    <none>        8000/TCP         9m30s
loksai-ui-svc    NodePort    10.110.213.143   <none>        8501:30007/TCP   9m29s
root@kmaster-node:~/kubernetes/actions-runner# 

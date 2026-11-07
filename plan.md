# GitOps-проєкт PetClinic: план і чеклист

Мета: повний потік GitHub → Jenkins → образ → ArgoCD → розгортання в кластері.

Стан на 7.11.2026. Позначки: `[x]` зроблено, `[ ]` ще ні.

## Основна частина

### 1. Кластер k3d
- [x] Кластер `my-cluster`: 1 server + 2 agents
- [x] Traefik вимкнений
- [x] Порти 80 і 443 відкриті (через `serverlb`)

### 2. ArgoCD через Terraform
- [x] `providers.tf` (kubernetes і helm)
- [x] `variables.tf`
- [x] `main.tf` (helm_release, namespace `argocd`)
- [x] `outputs.tf`
- [x] ArgoCD працює, вхід через `port-forward`

### 3. Інтеграція ArgoCD з GitHub (метод: PAT)
- [x] Репозиторій `l-lisssa/1petclinic-gitops` на GitHub
- [x] Токен тільки для читання, один репозиторій
- [x] Репозиторій підключений в ArgoCD (Settings → Repositories)

### 4. App-of-Apps
- [x] `bootstrap/root-app.yaml`
- [x] Папка `argocd-apps/`
- [x] Застосунок `root` стежить за папкою

### 5. NGINX Ingress і Jenkins через ArgoCD
- [x] `argocd-apps/nginx-ingress.yaml`
- [x] `argocd-apps/jenkins.yaml`
- [x] Jenkins відкривався за http://jenkins.localhost
- [ ] Jenkins зараз упав після перезапуску (init: CrashLoopBackOff), треба `kubectl delete pod jenkins-0 -n jenkins`

### 6. Форк і CI/CD (Jenkins)
- [x] Форк `l-lisssa/spring-petclinic`
- [x] Dockerfile і Jenkinsfile у форку
- [x] Токен Docker Hub (Read & Write)
- [x] Токен GitHub для Jenkins (Contents: Read and write)
- [x] Credentials у Jenkins: `dockerhub`, `github-gitops`
- [x] Завдання `spring-petclinic` (Pipeline from SCM)
- [x] Збірка #1: Finished SUCCESS
- [x] Образ на Docker Hub: `lizakalinichenko/spring-petclinic`
- [x] Jenkins записує tag у `charts/petclinic/values.yaml`
- [ ] Автозапуск збірки при коміті (зараз тільки Build Now)

### 7. База даних (CrunchyData Postgres)
- [x] `argocd-apps/pgo-registry.yaml` (OCI-реєстр)
- [x] `argocd-apps/pgo-operator.yaml` (чарт pgo 6.0.3)
- [x] `argocd-apps/petclinic-db.yaml`
- [x] `manifests/petclinic-db/postgres.yaml` (PostgresCluster)
- [ ] Підтвердити: застосунки `pgo-operator` і `petclinic-db` Healthy
- [ ] Підтвердити: поди в namespace `petclinic` Running
- [ ] Підтвердити: є Secret `petclinic-db-pguser-petclinic`

### 8. Helm-чарт PetClinic
- [ ] `charts/petclinic/Chart.yaml`
- [ ] `templates/deployment.yaml` (образ з values, змінні БД з Secret)
- [ ] `templates/service.yaml`
- [ ] `templates/ingress.yaml` (petclinic.localhost)
- [ ] Профіль Postgres у PetClinic (SPRING_PROFILES_ACTIVE=postgres)
- [ ] `argocd-apps/petclinic.yaml`
- [ ] PetClinic відкривається в браузері

### 9. Моніторинг
- [ ] kube-prometheus-stack через ArgoCD
- [ ] Grafana через Ingress (grafana.localhost)
- [ ] Метрики PetClinic (actuator/prometheus)

## Бонуси

### 10. cert-manager і TLS
- [ ] cert-manager через ArgoCD
- [ ] Самопідписаний CA для .localhost
- [ ] TLS для ArgoCD, Jenkins, PetClinic, Grafana

### 11. Gateway API замість Ingress
- [ ] Gateway + HTTPRoute для всіх застосунків

### 12. Elasticsearch / OpenSearch
- [ ] Розгортання через ArgoCD
- [ ] Збір логів кластера і PetClinic
- [ ] Інтеграція з Grafana
- [ ] Панель логів застосунку і панель логів кластера

### 13. Vault
- [ ] Vault через ArgoCD
- [ ] Креденшіали БД у Vault KV
- [ ] Доставка секретів у PetClinic (CSI, Operator або External Secrets)
- [ ] Kubernetes auth і політики з мінімальними правами
- [ ] Перевірка ротації секрету

## Фінальна перевірка (Expected Result)
- [ ] Коміт у форку → Jenkins збирає → новий tag у `1petclinic-gitops` → ArgoCD оновлює → PetClinic перезапускається

## Корисні команди

- Перед роботою з репозиторієм (Jenkins додає коміти): `git pull`
- Пароль ArgoCD: `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo`
- Тунель до ArgoCD: `kubectl port-forward svc/argocd-server -n argocd 8080:443` (https://localhost:8080, логін admin)
- Пароль Jenkins: `kubectl get secret -n jenkins jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 -d; echo` (логін admin)
- Jenkins не стартує (init Crash): `kubectl delete pod jenkins-0 -n jenkins`
- Кластер зупинений: `k3d cluster start my-cluster`
- Стан застосунків: `kubectl get applications -n argocd`

## Адреси

- ArgoCD: https://localhost:8080 (через port-forward)
- Jenkins: http://jenkins.localhost
- Репозиторій GitOps: https://github.com/l-lisssa/1petclinic-gitops
- Форк PetClinic: https://github.com/l-lisssa/spring-petclinic
- Образ: https://hub.docker.com/r/lizakalinichenko/spring-petclinic

## Структура репозиторію 1petclinic-gitops

- `terraform/`: ArgoCD через Terraform
- `bootstrap/root-app.yaml`: «менеджер» App-of-Apps
- `argocd-apps/`: описи застосунків (NGINX, Jenkins, pgo, база)
- `manifests/petclinic-db/`: заявка на базу
- `charts/petclinic/`: Helm-чарт PetClinic (поки лише values.yaml)
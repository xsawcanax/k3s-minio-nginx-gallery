# Mini-klaster K3s z MinIO i frontendem Nginx

## Cel projektu

Celem projektu jest uruchomienie lokalnego, wielowęzłowego klastra Kubernetes (k3s) i wdrożenie w nim:

* **Ingress Controller (nginx-ingress)** – do zarządzania ruchem HTTP i kierowania do aplikacji
* **MinIO** – obiektowy storage kompatybilny z S3, przechowujący obrazki
* **nginx-frontend** – serwis statyczny wyświetlający galerię obrazków z MinIO

Projekt pokazuje umiejętności konfiguracji klastra, wdrożenia aplikacji za pomocą Helm oraz konfiguracji Ingress i port-forwardingu.

---

## Skład środowiska

* k3d/k3s – lokalny klaster Kubernetes
* Rancher Desktop – zarządzanie klastrem
* MinIO – obiektowy storage
* Nginx (Bitnami) – serwer frontendowy
* Helm – do instalacji aplikacji
* mc (MinIO Client) – zarządzanie bucketami i politykami
* nginx-ingress Controller – zarządzanie ruchem HTTP

---

## 1. Utworzenie klastra k3s

Stwórz lokalny klaster z 1 serwerem i 2 agentami:

```bash
k3d cluster create moj-klaster --agents 2 --api-port 6550
kubectl get nodes
```

---

## 2. Instalacja nginx-ingress Controller

Dodaj repozytorium i zainstaluj kontroler:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx --create-namespace --namespace ingress-nginx
```

---

## 3. Instalacja MinIO

Dodaj repozytorium Bitnami i zainstaluj MinIO z własnym plikiem wartości:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install minio bitnami/minio -f minio/values-minio.yaml
```

Po instalacji sprawdź serwisy i zmień typ na NodePort, jeśli to konieczne:

```bash
kubectl get svc -l app.kubernetes.io/name=minio
kubectl patch svc minio --type merge -p '{"spec": {"type": "NodePort"}}'
```

---

## 4. Port-forwarding MinIO

Uzyskaj dostęp do MinIO GUI i API lokalnie:

```bash
kubectl port-forward svc/minio 9000:9000
kubectl port-forward svc/minio-console 9090:9090
```

Dostęp:

* GUI MinIO: `http://localhost:9090`
* API MinIO: `http://localhost:9000`

---

## 5. Konfiguracja MinIO Client (mc)

Dodaj alias do klienta i ustaw publiczny dostęp do bucketa `demo`:

```bash
mc alias set local http://localhost:9000 minioadmin minioadmin
mc anonymous set download local/demo
```

---

## 6. Dodanie obrazków do MinIO

Zaloguj się do GUI MinIO i wgraj obrazki do bucketa `demo`.

---

## 7. Utworzenie ConfigMap dla frontendu

Utwórz ConfigMap `nginx-index` z plikiem `index.html` lokalnie lub z repozytorium:

```bash
kubectl create configmap nginx-index --from-file=frontend/index.html
```

---

## 8. Instalacja frontendu nginx-frontend

Użyj pliku `values-frontend.yaml` (w repozytorium) do konfiguracji wolumenów, punktów montowania, portów i zmiennych środowiskowych.

Zainstaluj lub zaktualizuj frontend:

```bash
helm install nginx-frontend bitnami/nginx -f frontend/values-frontend.yaml
# lub
helm upgrade nginx-frontend bitnami/nginx -f frontend/values-frontend.yaml
```

---

## 9. Konfiguracja Ingress

Stwórz i zastosuj plik `ingress.yaml` dla przekierowania ruchu HTTP do serwisu frontendowego:

```bash
kubectl apply -f ingress/ingress.yaml
```

---

## 10. Sprawdzenie działania

Otwórz przeglądarkę pod adresem:

```
http://localhost:8080
```

Powinna się wyświetlić galeria obrazków z MinIO.

---
## Autoskalowanie nginx-frontend

1. Sprawdź działanie Metrics Server:

```bash
kubectl top nodes
kubectl top pods
```

2. Zastosuj manifest HPA (plik `nginx-frontend-hpa.yaml`):

```bash
kubectl apply -f nginx-frontend-hpa.yaml
```

3. Monitoruj autoskalera:

```bash
kubectl get hpa nginx-frontend-hpa
kubectl describe hpa nginx-frontend-hpa
```

---
## Testowanie działania autoskalera

Aby przetestować, czy autoskaler działa poprawnie, możesz użyć narzędzia `hey` do generowania ruchu HTTP obciążającego serwis.

### Instalacja hey

Pobierz binarkę `hey` odpowiednią dla swojego systemu ze [strony projektu](https://github.com/rakyll/hey/releases).

### Uruchomienie testu obciążeniowego

Uruchom poniższe polecenie, które przez 2 minuty będzie generować 50 zapytań na sekundę, z 50 równoczesnymi klientami:

```bash
./hey.exe -z 2m -q 50 -c 50 http://localhost:8080/
```

### Co to robi?

* `-z 2m` — test trwa 2 minuty
* `-q 50` — 50 zapytań na sekundę
* `-c 50` — 50 równoczesnych klientów

Obciążenie to powinno zwiększyć wykorzystanie CPU podów, co spowoduje skalowanie się deploymentu dzięki HPA.



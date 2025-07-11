# Mini-klaster K3s z MinIO i frontendem Nginx

## Skład środowiska

- **k3d/k3s** – lokalny klaster Kubernetes
- **Rancher Desktop** – zarządzanie klastrem
- **MinIO** – obiektowy storage (jak S3), hostujący obrazki
- **Nginx (Bitnami)** – serwer frontendowy
- **Helm** – do instalacji aplikacji
- **mc (MinIO Client)** – zarządzanie bucketami i politykami

---

## Uruchomienie klastra

```bash
k3d cluster create moj-klaster --agents 2
```

---

## Instalacja MinIO

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm install minio bitnami/minio -f minio/values-minio.yaml
```

Po instalacji sprawdź:

```bash
kubectl get svc -l app.kubernetes.io/name=minio
```

Zmieniamy typ serwisu `minio` na `NodePort` (jeśli nie jest ustawiony):

```bash
kubectl patch svc minio --type merge -p '{"spec": {"type": "NodePort"}}'
```

---

## Dostęp do MinIO

- GUI: http://localhost:9090  
- API / zdjęcia: http://localhost:31200

Dane logowania (domyślne):

- **Login:** `minioadmin`
- **Hasło:** `minioadmin`

---

## Dodawanie obrazków do MinIO

1. Wejdź w GUI: http://localhost:9090
2. Zaloguj się.
3. Utwórz bucket o nazwie: `demo`
4. Wgraj tam obrazki (`image1.jpg`, `image2.jpg`, `image3.jpg`)

Uwaga: pliki powinny być dostępne pod adresem:  
`http://localhost:31200/demo/image1.jpg` itd.

---

## Ustawienie polityki publicznej

```bash
./mc alias set local http://localhost:31200 minioadmin minioadmin
./mc anonymous set download local/demo
```

---

## Instalacja frontendu (NGINX)

```bash
helm install nginx-frontend bitnami/nginx -f frontend/values-frontend.yaml
```

Frontend używa `ConfigMap` o nazwie `nginx-index`, zawierającej statyczny `index.html`.

---

## Zawartość index.html (ConfigMap)

```html
<!DOCTYPE html>
<html lang="pl">
<head>
<meta charset="UTF-8">
<title>Galeria</title>
<style>
body { font-family: Arial, sans-serif; text-align: center; padding: 20px; }
h1 { margin-bottom: 20px; }
.gallery { display: flex; justify-content: center; gap: 15px; flex-wrap: wrap; }
.gallery img { max-width: 300px; border: 2px solid #ddd; border-radius: 8px; }
</style>
</head>
<body>
<h1>Galeria z MinIO</h1>
<div class="gallery">
  <img src="http://localhost:31200/demo/image1.jpg" alt="Image 1">
  <img src="http://localhost:31200/demo/image2.jpg" alt="Image 2">
  <img src="http://localhost:31200/demo/image3.jpg" alt="Image 3">
</div>
</body>
</html>
```

---

## Struktura katalogów projektu

```
projekt/
├── frontend/
│   ├── values-frontend.yaml
│   └── nginx-deployment.yaml (opcjonalnie)
├── minio/
│   └── values-minio.yaml
├── ingress/
│   └── ingress.yaml (opcjonalnie)
├── mc.exe (MinIO client)
└── README.md
```

---

## Sprawdzenie działania

Otwórz przeglądarkę i przejdź do:

```
http://localhost:8080
```

Jeśli wszystko działa poprawnie, zobaczysz galerię z obrazkami z MinIO

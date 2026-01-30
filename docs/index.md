# Python App (Flask)

Pequeña API en Flask que expone dos endpoints para ver información del pod y un chequeo de salud. Este documento cubre cómo ejecutarla localmente, en contenedor, y desplegarla en Kubernetes (manifiestos y Helm), además de describir los endpoints disponibles.

## Resumen

- Framework: Flask
- Puerto por defecto: 5000
- Endpoints:
	- `GET /api/v1/healthz` — estado de salud
	- `GET /api/v1/info` — hora, hostname y mensaje
- Código principal: ver [python-app/src/app.py](../src/app.py)

## Endpoints

### `GET /api/v1/healthz`

Chequeo de salud simple.

Ejemplo:

```bash
curl -s http://localhost:5000/api/v1/healthz
```

Respuesta:

```json
{"status":"up"}
```

### `GET /api/v1/info`

Devuelve información del sistema y un mensaje.

Ejemplo:

```bash
curl -s http://localhost:5000/api/v1/info | jq
```

Respuesta típica:

```json
{
	"time": "10:11:12AM on January 30, 2026",
	"hostname": "python-app-7f5c9c9b5b-2x7m8",
	"message": "You are doing well!! :)",
	"deployed": "on k8s"
}
```

## Ejecución local

Requisitos: Python 3.12+ y `pip`.

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python src/app.py
```

Prueba rápida:

```bash
curl http://localhost:5000/api/v1/healthz
curl http://localhost:5000/api/v1/info
```

## Contenedor (Docker/Podman)

La imagen se construye a partir de [python-app/Dockerfile](../Dockerfile).

Construir y ejecutar:

```bash
# Docker
docker build -t python-app:local .
docker run --rm -p 5000:5000 python-app:local

# Podman
podman build -t python-app:local .
podman run --rm -p 5000:5000 python-app:local
```

## Despliegue en Kubernetes (manifiestos)

Manifiestos en [python-app/k8s/](../k8s):

- Deployment: usa la imagen `clloris/python-app:v2-1` y expone `containerPort: 5000`.
- Service: `ClusterIP` en el puerto `8080` apuntando a `targetPort: 5000`.
- Ingress: clase `nginx`, host `python-app.test.com`.

Aplicar:

```bash
kubectl apply -f k8s/deploy.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml
```

Probar (asumiendo Ingress con DNS/resolución adecuada):

```bash
curl http://python-app.test.com/api/v1/healthz
curl http://python-app.test.com/api/v1/info
```

## Despliegue con Helm

Chart en [python-app/charts/python-app](../charts/python-app).

Valores relevantes (por defecto):

- `service.port`: 5000 (coincide con el puerto del contenedor)
- `ingress.enabled`: true, `ingress.className`: `nginx`, host `python-app.test.com`
- Probes `liveness/readiness`: `GET /api/v1/healthz`

Instalación básica:

```bash
helm install python-app ./charts/python-app \
	--set image.repository=clloris/python-app \
	--set image.tag=ee70802
```

Actualizar valores (ejemplo para cambiar host):

```bash
helm upgrade python-app ./charts/python-app \
	--set ingress.hosts[0].host=myapp.local
```

## Troubleshooting

- El Service publica `8080` pero el contenedor escucha en `5000`; verifica que tu Ingress apunte al Service en el puerto `8080`.
- Comprueba salud con `curl <host>/api/v1/healthz` y revisa logs del pod si falla.
- Si usas otro IngressClass, ajusta `ingress.className` en `values.yaml`.

## Estructura

- Código: [python-app/src/app.py](../src/app.py)
- Requisitos: [python-app/requirements.txt](../requirements.txt)
- Dockerfile: [python-app/Dockerfile](../Dockerfile)
- Manifiestos: [python-app/k8s/](../k8s)
- Helm Chart: [python-app/charts/python-app](../charts/python-app)


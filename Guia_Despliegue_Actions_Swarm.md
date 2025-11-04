# Guía completa: GitHub Actions + Docker Hub + AWS (4 EC2 en Docker Swarm con 10 réplicas)  

> **Objetivo**: Construir y publicar una imagen Docker de una API Node/Express en **Docker Hub** con **GitHub Actions**, y desplegarla en **AWS** sobre **Docker Swarm** con **4 instancias EC2** (1 líder + 3 managers) corriendo **10 réplicas**, verificando por **IP pública** y entregando pantallazos.

---

## B. Cuentas y tokens (crear TODO desde cero)

### B.1 GitHub
1. Ir a https://github.com → **Sign up** → crear cuenta y verificar correo.

### B.2 Docker Hub
1. Ir a https://hub.docker.com → **Sign up** → crear cuenta **<DOCKERHUB_USER>**.

### B.3 Docker Hub Personal Access Token (para GitHub Actions)
1. Docker Hub → avatar (arriba derecha) → **Settings**.  
2. **Personal access tokens** → **New access token**.  
3. **Description**: `github-actions` | **Expires**: `Never` | **Permissions**: `Read & Write`.  
4. **Generate** → **copiar** el token (se muestra una sola vez).

> Guardar este token. Se pegará en **GitHub → Secrets** como `DOCKERHUB_TOKEN`.

---

## C. Preparar el proyecto Node (mínimo viable)

Crea una carpeta `node-arbitro-api` y pon los archivos que siguen. Si ya tienen un ZIP, asegúrense de que al menos exista `/health` y un `Dockerfile` válido.

### C.2 `src/server.js`
```js
// MODIFICADOOOOOOOOOOOO ULTIMOOOOOOOOOO
import express from "express";
import cors from "cors";
import dotenv from "dotenv";

// === AGREGADO (para evidencias de réplicas en Swarm) ===
import os from "os"; // <-- AGREGADO: usar hostname del contenedor

import arbitroRoutes from "./routes/arbitro.js";
import authRoutes from "./routes/auth.js";
import newsRoutes from "./routes/news.js";

// ============================
// Swagger
// ============================
import swaggerUi from "swagger-ui-express"; // Paquete para servir Swagger UI
import swaggerSpec from "./swagger.js";      // Tu configuración Swagger (src/swagger.js)

// ⬇️ NUEVO: para servir el customJs (interceptor PDF)
import path from "path";
import { fileURLToPath } from "url";

dotenv.config();

const app = express();
const PORT = process.env.PORT || 4000;

// Para resolver rutas de /public
const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

// ============================
// Middlewares
// ============================
app.use(cors());
app.use(express.json({ limit: "2mb" }));

// (opcional) Soportar x-www-form-urlencoded si algún cliente lo usa
app.use(express.urlencoded({ extended: true, limit: "2mb" }));

// === AGREGADO (para evidencias de balanceo) ===============================
// Inyecta en TODAS las respuestas el ID del contenedor (hostname)
// Permite que el profe vea contenedores distintos en llamadas sucesivas
app.use((req, res, next) => {
  res.set("x-container-id", os.hostname()); // <-- AGREGADO
  next();
});
// ==========================================================================

// ============================
// Rutas base
// ============================
app.get("/", (_req, res) =>
  res.json({
    ok: true,
    name: "node-arbitro-api",
    version: "1.0.0",
    containerId: os.hostname(), // <-- AGREGADO (opcional pero útil para pantallazo)
  })
);

// === AGREGADO (endpoint de salud para pantallazos) =========================
app.get("/health", (_req, res) => {
  res.json({
    ok: true,
    containerId: os.hostname(), // evidencia de réplica
    now: new Date().toISOString(),
  });
});
// ==========================================================================

app.use("/api/auth", authRoutes);
app.use("/api/arbitro", arbitroRoutes);
app.use("/api/news", newsRoutes);

// ============================
// Servir estáticos (customJs)
// ============================
// Coloca tu archivo "public/swagger-pdf.js" en la raíz del proyecto (al nivel de /src)
app.use("/assets", express.static(path.join(__dirname, "../public")));

// ============================
// Swagger UI
// ============================
// Esto expone la documentación interactiva en:
// 👉 http://localhost:4000/api/docs
app.use(
  "/api/docs",
  swaggerUi.serve,
  swaggerUi.setup(swaggerSpec, {
    swaggerOptions: {
      persistAuthorization: true,       // recuerda el token entre recargas
      displayRequestDuration: true,
    },
    // ⬇️ NUEVO: inyecta script que intercepta respuestas PDF y fuerza descarga .pdf
    customJs: "/assets/swagger-pdf.js",
  })
);

// ============================
// Manejo de rutas no encontradas
// ============================
app.use((req, res) => {
  res.status(404).json({
    ok: false,
    error: "Not Found",
    path: req.originalUrl,
  });
});

// ============================
// Servidor
// ============================
app.listen(PORT, () => {
  console.log(`[node-arbitro-api] Listening on http://localhost:${PORT}`);
  console.log(`Swagger UI disponible en http://localhost:${PORT}/api/docs`);
});

```

### C.3 `Dockerfile`
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
ENV NODE_ENV=production
ENV PORT=4000
EXPOSE 4000
CMD ["node","src/server.js"]
```

> La API **no** debe depender de BD local para `/health`. El resto (login, etc.) se integrará luego con Spring/MySQL o RDS.

---

## D. Subir a GitHub (repo nuevo)

### D.1 Crear repositorio
1. GitHub (web) → **New repository** → **Repository name**: `node-arbitro-api` → **Public** → **Create repository**.

### D.2 Publicar código desde tu PC
En PowerShell dentro de la carpeta del proyecto:
```powershell
git init
git add .
git commit -m "init: node arbitro api base"
git branch -M main
git remote add origin https://github.com/<GITHUB_USER>/node-arbitro-api.git
git push -u origin main
```

> Reemplaza `<GITHUB_USER>` por tu usuario.

---

## E. GitHub Actions que publica a Docker Hub

### E.1 Agregar Secret `DOCKERHUB_TOKEN`
- Repo → **Settings → Secrets and variables → Actions → New repository secret**  
  - **Name**: `DOCKERHUB_TOKEN`  
  - **Secret**: (pega el token generado en Docker Hub) → **Add secret**.

### E.2 Workflow `.github/workflows/docker-publish.yml`
```yaml
name: Publish Docker image
on:
  push:
    branches: [ "main" ]
  workflow_dispatch:

env:
  IMAGE_NAME: <DOCKERHUB_USER>/arbitros-api  # <-- cambia esto

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: <DOCKERHUB_USER>               # <-- cambia
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.IMAGE_NAME }}
          tags: |
            type=raw,value=latest
            type=sha

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
```

### E.3 Ejecutar y verificar
- Haz un **commit** a `main` o usa **Actions → Run workflow**.  
- Verifica en **Actions** → job en **verde** (Success).  
- Verifica en **Docker Hub** → repo `<DOCKERHUB_USER>/arbitros-api` con tag **`latest`**.

**Pantallazos a entregar (CI/CD):**
- ✅ Actions con el job **Success**  
- ✅ Docker Hub con repo y tag **latest**

---

## F. AWS: 4 EC2 con Docker + Security Group

### F.1 Security Group `swarm-sg`
- EC2 → **Security Groups** → **Create security group**  
  - Name: `swarm-sg`  
  - VPC: *default*  
  - **Inbound rules**:
    - HTTP **80/TCP** → `0.0.0.0/0`
    - Custom TCP **2377** → `0.0.0.0/0`
    - Custom TCP **7946** → `0.0.0.0/0`
    - Custom UDP **7946** → `0.0.0.0/0`
    - Custom UDP **4789** → `0.0.0.0/0`
    - *(Opcional)* SSH **22/TCP** → **My IP**
  - **Outbound**: All traffic → **Create**.

**Pantallazo a entregar (SG):**  
- ✅ Inbound rules con 80/2377/7946 (tcp/udp)/4789 (udp).

### F.2 Lanzar 4 instancias Ubuntu (Instance Connect)
Repite **4 veces**:
1. EC2 → **Launch instances**  
2. **Name**: `swarm-leader` (luego `swarm-mgr-2`, `swarm-mgr-3`, `swarm-mgr-4`)  
3. **AMI**: Ubuntu Server **24.04 LTS (amd64)**  
4. **Instance type**: `t3.micro`  
5. **Key pair**: *Proceed without a key pair* (o selecciona uno si usarás SSH)  
6. **Network settings**:
   - VPC: default  
   - Subnet: **No preference**  
   - Auto-assign public IP: **Enable**  
   - **Security group**: **`swarm-sg`**
7. **Advanced → User data** (instala Docker):
   ```yaml
   #cloud-config
   package_update: true
   package_upgrade: true
   runcmd:
     - curl -fsSL https://get.docker.com | sh
     - usermod -aG docker ubuntu
     - systemctl enable docker
     - systemctl start docker
     - echo "Docker ready" > /var/tmp/docker-ready
   ```
8. **Launch instance**. Espera **2/2 checks**.

**Pantallazo a entregar (EC2):**  
- ✅ Lista de **4 instancias Running** con SG `swarm-sg` y **Public IPv4**.

---

## G. Formar Docker Swarm (1 líder + 3 managers)

### G.1 Conectar a la **líder**
- EC2 → `swarm-leader` → **Connect → EC2 Instance Connect → Connect**  
- Verificar Docker:
  ```bash
  docker version
  ```

### G.2 Inicializar Swarm
```bash
hostname -I   # IP privada 172.31.x.x
docker swarm init --advertise-addr <IP_PRIVADA_LIDER>
docker swarm join-token manager
```
(Copiar el `docker swarm join --token ...`)

### G.3 Unir 3 managers
En **cada** `swarm-mgr-2`, `swarm-mgr-3`, `swarm-mgr-4`:
```bash
docker swarm join --token <TOKEN_MANAGER> <IP_PRIVADA_LIDER>:2377
```

### G.4 Ver nodos (en la líder)
```bash
docker node ls
```
Debe mostrar **1 Leader + 3 Reachable**.

**Pantallazo a entregar (Swarm):**  
- ✅ `docker node ls` (Leader + 3 Reachable).

---

## H. Desplegar la API con 10 réplicas (Stack)

### H.1 `stack.yml` (en la líder)
```bash
cat > stack.yml <<'YAML'
version: "3.8"

services:
  api:
    image: <DOCKERHUB_USER>/arbitros-api:latest
    ports:
      - "80:4000"
    environment:
      NODE_ENV: production
      PORT: "4000"
    deploy:
      replicas: 10
      update_config:
        order: start-first
        parallelism: 2
      restart_policy:
        condition: on-failure
    networks: [public]

networks:
  public:
    driver: overlay
YAML
```

### H.2 Crear red y desplegar
```bash
docker network create -d overlay public || true
docker stack deploy -c stack.yml arbitros
```

### H.3 Ver estado
```bash
docker service ls
docker service ps arbitros_api
```
Debe verse `arbitros_api` **10/10** y tasks **Running**.

**Pantallazos a entregar (Servicios):**  
- ✅ `docker service ls` (10/10).  
- ✅ `docker service ps arbitros_api` (Running).

---

## I. Pruebas por IP pública (balanceo y Swagger)

### I.1 IP pública
- EC2 → copia **Public IPv4** (de la líder, por ejemplo).

### I.2 Probar `/health` (Windows PowerShell)
```powershell
curl.exe -i http://<IP_PUBLICA>/health
curl.exe -i http://<IP_PUBLICA>/health
curl.exe -i http://<IP_PUBLICA>/health
```
Debe cambiar **`x-container-id`** (o `containerId` en JSON) entre llamadas.

### I.3 (Opcional) Swagger
```
http://<IP_PUBLICA>/api/docs
```

**Pantallazos a entregar (Pruebas):**  
- ✅ 2–3 `curl -i /health` con IDs distintos.  
- ✅ (Opcional) Swagger UI.

---

## J. Operación

- **Escalar**:
  ```bash
  docker service scale arbitros_api=15
  ```
- **Actualizar imagen**:
  ```bash
  docker pull <DOCKERHUB_USER>/arbitros-api:latest
  docker stack deploy -c stack.yml arbitros
  ```
- **Logs / diagnóstico**:
  ```bash
  docker service logs -f arbitros_api
  docker service ps --no-trunc arbitros_api
  ```
- **Eliminar**:
  ```bash
  docker stack rm arbitros
  ```

---

## K. Checklist de entrega (pantallazos)

1) **CI/CD**: Actions verde + Docker Hub con `latest`  
2) **Infra**: SG `swarm-sg` mostrando 80/2377/7946 tcp+udp / 4789 udp + 4 EC2 Running  
3) **Swarm**: `docker node ls` (Leader + 3 Reachable)  
4) **Servicio**: `docker service ls` (10/10) + `docker service ps arbitros_api` (Running)  
5) **Pruebas**: 2–3 `curl -i /health` con IDs distintos + (opcional) Swagger

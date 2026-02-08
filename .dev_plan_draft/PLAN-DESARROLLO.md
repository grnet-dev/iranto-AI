# Plan de Desarrollo - Infraestructura Docker + GitHub

## 📋 Resumen del Proyecto

Desarrollo de un ecosistema de aplicaciones internas con las siguientes características:
- **3 desarrolladores**: 1 Windows (Antigravity), 2 Linux laptops
- **Servidor Linux** con Docker para servicios core
- **VPN Fortinet** para acceso remoto
- **GitHub** como plataforma de control de versiones
- **Automatización completa** con GitHub Actions

## 🏗️ Arquitectura del Sistema

### Stack Tecnológico
- **Frontend**: SvelteKit (múltiples apps)
- **Backend**: Node.js (API REST)
- **Servicios Core**: Python (Graph-RAG, LLM, auditoría)
- **Base de Datos**: ArangoDB
- **Cache**: Redis
- **AI/LLM**: Ollama
- **Proxy**: Nginx
- **Contenedores**: Docker + Docker Compose

### Entornos
- **Desarrollo**: Servidor Linux local + VPN Fortinet
- **Producción**: Servidor con acceso internet + Nginx SSL

## 📁 Estructura del Repositorio

```
empresa-infra/
├── .github/
│   └── workflows/
│       ├── ci-develop.yml          # Pipeline desarrollo
│       ├── ci-main.yml             # Pipeline producción
│       └── docker-build.yml        # Build Docker images
├── apps/                           # Aplicaciones SvelteKit/Node.js
│   ├── admin-settings/
│   ├── inventario/
│   ├── seguridad/
│   ├── proyectos/
│   ├── fabricacion/
│   └── rrhh/
├── services/                       # Servicios Python
│   ├── graph-rag/
│   ├── llm-auditoria/
│   └── utils-backend/
├── docker/                         # Configuraciones Docker
│   ├── development/
│   │   └── docker-compose.dev.yml
│   └── production/
│       └── docker-compose.prod.yml
├── nginx/
│   ├── dev.conf
│   └── prod.conf
├── scripts/                        # Scripts de desarrollo
│   ├── dev-setup.sh
│   ├── backup.sh
│   └── health-check.sh
├── shared/                         # Componentes compartidos
│   └── common-types/
├── docs/                           # Documentación
├── .env.example                    # Variables de entorno ejemplo
├── .gitignore
├── README.md
└── package.json                    # Scripts monorepo
```

## 🐳 Configuración Docker

### Docker Compose Desarrollo
```yaml
# docker/development/docker-compose.dev.yml
version: '3.8'

services:
  # Base de Datos Principal
  arangodb:
    image: arangodb:latest
    container_name: empresa-arangodb
    ports:
      - "8529:8529"
    environment:
      ARANGO_ROOT_PASSWORD: ${ARANGO_PASSWORD}
      ARANGO_NO_AUTH: "false"
    volumes:
      - arango_data:/var/lib/arangodb3
      - ./docker/arangodb:/docker-entrypoint-initdb.d
    networks:
      - empresa-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8529/_api/version"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Cache y Session Store
  redis:
    image: redis:alpine
    container_name: empresa-redis
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    networks:
      - empresa-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Servidor LLM
  ollama:
    image: ollama/ollama:latest
    container_name: empresa-ollama
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    networks:
      - empresa-network
    restart: unless-stopped
    environment:
      - OLLAMA_HOST=0.0.0.0
    deploy:
      resources:
        limits:
          memory: 8G
        reservations:
          memory: 4G

  # Reverse Proxy (Desarrollo)
  nginx:
    image: nginx:alpine
    container_name: empresa-nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx/dev.conf:/etc/nginx/nginx.conf:ro
      - nginx_logs:/var/log/nginx
    networks:
      - empresa-network
    depends_on:
      - arangodb
      - redis
    restart: unless-stopped

volumes:
  arango_data:
    driver: local
  redis_data:
    driver: local
  ollama_data:
    driver: local
  nginx_logs:
    driver: local

networks:
  empresa-network:
    driver: bridge
```

### Docker Compose Producción
```yaml
# docker/production/docker-compose.prod.yml
version: '3.8'

services:
  arangodb:
    image: arangodb:latest
    container_name: empresa-arangodb-prod
    environment:
      ARANGO_ROOT_PASSWORD: ${ARANGO_PASSWORD_PROD}
    volumes:
      - arango_prod_data:/var/lib/arangodb3
      - ./backups:/backups
    networks:
      - empresa-network
    restart: always
    # No exponer puertos en producción, acceso interno únicamente

  redis:
    image: redis:alpine
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD_PROD}
    volumes:
      - redis_prod_data:/data
    networks:
      - empresa-network
    restart: always

  ollama:
    image: ollama/ollama:latest
    volumes:
      - ollama_prod_data:/root/.ollama
    networks:
      - empresa-network
    restart: always
    deploy:
      resources:
        limits:
          memory: 12G
        reservations:
          memory: 6G

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/prod.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
      - nginx_prod_logs:/var/log/nginx
    networks:
      - empresa-network
    restart: always

volumes:
  arango_prod_data:
  redis_prod_data:
  ollama_prod_data:
  nginx_prod_logs:

networks:
  empresa-network:
    driver: bridge
```

## 🔄 GitHub Actions - CI/CD

### Pipeline Desarrollo
```yaml
# .github/workflows/ci-develop.yml
name: CI Development

on:
  push:
    branches: [develop]
  pull_request:
    branches: [develop]

jobs:
  test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [18.x, 20.x]
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
          
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          
      - name: Install dependencies
        run: |
          npm ci
          pip install -r requirements.txt
          
      - name: Run tests
        run: |
          npm run test:all
          python -m pytest services/ --cov=services/
          
      - name: Build applications
        run: |
          npm run build:all
          
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run CodeQL
        uses: github/codeql-action/init@v2
        with:
          languages: javascript, python
          
  deploy-dev:
    needs: [test, security-scan]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    
    steps:
      - name: Deploy to development server
        uses: appleboy/ssh-action@v0.1.5
        with:
          host: ${{ secrets.DEV_SERVER_HOST }}
          username: ${{ secrets.DEV_SERVER_USER }}
          key: ${{ secrets.DEV_SERVER_SSH_KEY }}
          script: |
            cd /opt/empresa-infra
            git pull origin develop
            docker-compose -f docker/development/docker-compose.dev.yml down
            docker-compose -f docker/development/docker-compose.dev.yml up -d --build
            docker system prune -f
```

### Pipeline Producción
```yaml
# .github/workflows/ci-main.yml
name: CI Production

on:
  push:
    branches: [main]

jobs:
  deploy-prod:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        
      - name: Deploy to production
        uses: appleboy/ssh-action@v0.1.5
        with:
          host: ${{ secrets.PROD_SERVER_HOST }}
          username: ${{ secrets.PROD_SERVER_USER }}
          key: ${{ secrets.PROD_SERVER_SSH_KEY }}
          script: |
            cd /opt/empresa-infra
            git pull origin main
            docker-compose -f docker/production/docker-compose.prod.yml down
            docker-compose -f docker/production/docker-compose.prod.yml up -d --build
            ./scripts/health-check.sh
            docker system prune -f
            
      - name: Backup databases
        run: |
          ssh ${{ secrets.PROD_SERVER_USER }}@${{ secrets.PROD_SERVER_HOST }} \
            "cd /opt/empresa-infra && ./scripts/backup.sh"
```

## 🔐 Configuración de Seguridad

### GitHub Secrets Requeridos
```bash
# Development Server
DEV_SERVER_HOST=192.168.x.x
DEV_SERVER_USER=deploy
DEV_SERVER_SSH_KEY=-----BEGIN OPENSSH PRIVATE KEY-----

# Production Server  
PROD_SERVER_HOST=servidor-produccion.com
PROD_SERVER_USER=deploy
PROD_SERVER_SSH_KEY=-----BEGIN OPENSSH PRIVATE KEY-----

# Application Secrets
ARANGO_PASSWORD=secure_password_dev
ARANGO_PASSWORD_PROD=secure_password_prod
REDIS_PASSWORD=redis_password_dev
REDIS_PASSWORD_PROD=redis_password_prod
```

### Configuración Firewall (Servidor Linux)
```bash
# UFW Rules
ufw allow ssh
ufw allow from 10.0.0.0/8 to any port 8529  # ArangoDB (VPN range)
ufw allow from 10.0.0.0/8 to any port 6379  # Redis (VPN range)
ufw allow from 10.0.0.0/8 to any port 11434 # Ollama (VPN range)
ufw allow 80/tcp                           # HTTP
ufw allow 443/tcp                          # HTTPS
ufw enable
```

## 💻 Configuración Local por Desarrollador

### Equipo Windows (Antigravity)
```batch
@echo off
REM setup-dev-windows.bat

SET SERVER_IP=192.168.1.100
SET ARANGO_URL=http://%SERVER_IP:8529
SET REDIS_URL=redis://%SERVER_IP:6379
SET OLLAMA_URL=http://%SERVER_IP:11434

echo Entorno de desarrollo configurado
echo Servidor: %SERVER_IP%
echo ArangoDB: %ARANGO_URL%
echo Redis: %REDIS_URL%
echo Ollama: %OLLAMA_URL%
```

### Laptops Linux
```bash
#!/bin/bash
# scripts/dev-setup-linux.sh

export SERVER_IP="192.168.1.100"
export ARANGO_URL="http://$SERVER_IP:8529"
export REDIS_URL="redis://$SERVER_IP:6379"
export OLLAMA_URL="http://$SERVER_IP:11434"

echo "Entorno de desarrollo configurado"
echo "Servidor: $SERVER_IP"
echo "ArangoDB: $ARANGO_URL"
echo "Redis: $REDIS_URL"
echo "Ollama: $OLLAMA_URL"

# Crear archivo .env local
cat > .env << EOF
SERVER_IP=$SERVER_IP
ARANGO_URL=$ARANGO_URL
REDIS_URL=$REDIS_URL
OLLAMA_URL=$OLLAMA_URL
EOF
```

## 🚀 Flujo de Trabajo de Desarrollo

### 1. Configuración Inicial
```bash
# Clonar repositorio
git clone https://github.com/empresa/empresa-infra.git
cd empresa-infra

# Configurar entorno local
chmod +x scripts/dev-setup.sh
./scripts/dev-setup.sh

# Instalar dependencias
npm install
pip install -r requirements.txt
```

### 2. Flujo de Desarrollo Diario
```bash
# 1. Sincronizar con main
git checkout main
git pull origin main

# 2. Crear rama de feature
git checkout -b feature/nueva-funcionalidad

# 3. Desarrollo local
# Editar archivos, hacer commits
git add .
git commit -m "feat: implementar nueva funcionalidad"

# 4. Push y PR automático
git push origin feature/nueva-funcionalidad

# 5. GitHub Actions automáticamente:
#    - Ejecuta tests
#    - Build y deploy a desarrollo
#    - Espera aprobación para producción
```

### 3. Merge a Producción
```bash
# Después de aprobación del PR
git checkout main
git pull origin main
git merge develop
git push origin main

# GitHub Actions automáticamente deploy a producción
```

## 📊 Monitoreo y Mantenimiento

### Health Checks Automáticos
```bash
#!/bin/bash
# scripts/health-check.sh

echo "=== Health Check Servicios ==="

# Verificar ArangoDB
if curl -f http://localhost:8529/_api/version > /dev/null 2>&1; then
    echo "✅ ArangoDB: OK"
else
    echo "❌ ArangoDB: ERROR"
fi

# Verificar Redis
if redis-cli -h localhost -p 6379 ping > /dev/null 2>&1; then
    echo "✅ Redis: OK"
else
    echo "❌ Redis: ERROR"
fi

# Verificar Ollama
if curl -f http://localhost:11434/api/tags > /dev/null 2>&1; then
    echo "✅ Ollama: OK"
else
    echo "❌ Ollama: ERROR"
fi

# Verificar Nginx
if curl -f http://localhost:80 > /dev/null 2>&1; then
    echo "✅ Nginx: OK"
else
    echo "❌ Nginx: ERROR"
fi
```

### Backup Automático
```bash
#!/bin/bash
# scripts/backup.sh

BACKUP_DIR="/opt/backups/empresa-infra"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# Backup ArangoDB
docker exec empresa-arangodb arangodump \
  --server.endpoint tcp://localhost:8529 \
  --server.username root \
  --server.password $ARANGO_PASSWORD_PROD \
  --output directory "/tmp/arangodb_backup_$DATE"

# Backup Redis
docker exec empresa-redis redis-cli --rdb /tmp/redis_backup_$DATE.rdb

# Backup Ollama models
docker run --rm -v ollama_prod_data:/data -v $BACKUP_DIR:/backup \
  alpine tar czf /backup/ollama_backup_$DATE.tar.gz -C /data .

echo "Backup completado: $DATE"
```

## 🛠️ Scripts de Utilidad

### Makefile
```makefile
# Makefile
.PHONY: install test build deploy-dev deploy-prod clean

install:
	npm install
	pip install -r requirements.txt

test:
	npm run test:all
	python -m pytest services/

build:
	npm run build:all

deploy-dev:
	./scripts/deploy.sh dev

deploy-prod:
	./scripts/deploy.sh prod

clean:
	docker system prune -f
	docker volume prune -f

logs-dev:
	docker-compose -f docker/development/docker-compose.dev.yml logs -f

logs-prod:
	docker-compose -f docker/production/docker-compose.prod.yml logs -f

health:
	./scripts/health-check.sh

backup:
	./scripts/backup.sh
```

## 📋 Checklist de Implementación

### Pre-requisitos Servidor
- [ ] Ubuntu 20.04+ o CentOS 8+
- [ ] Docker y Docker Compose instalados
- [ ] Git instalado
- [ ] SSH configurado
- [ ] Firewall UFW configurado
- [ ] Usuario 'deploy' con sudo sin password
- [ ] Conexión VPN Fortinet funcionando

### Pre-requisitos GitHub
- [ ] Repositorio creado
- [ ] GitHub Actions habilitados
- [ ] Secrets configurados
- [ ] Branch protection rules configuradas
- [ ] Dependabot habilitado
- [ ] Code scanning habilitado

### Pre-requisitos Desarrolladores
- [ ] Git configurado
- [ ] Node.js 18+ instalado
- [ ] Python 3.11+ instalado
- [ ] Acceso VPN a red local
- [ ] Editor/IDE configurado (Antigravity/VSCode)

### Implementación Paso a Paso
1. [ ] Configurar servidor Linux base
2. [ ] Crear repositorio GitHub
3. [ ] Configurar GitHub Actions y secrets
4. [ ] Implementar estructura de archivos
5. [ ] Configurar Docker Compose desarrollo
6. [ ] Implementar scripts de utilidad
7. [ ] Configurar Nginx y SSL
8. [ ] Testear pipeline de CI/CD
9. [ ] Onboarding de desarrolladores
10. [ ] Documentación y entrenamiento

## 🚨 Consideraciones Importantes

### Seguridad
- Nunca commitear secrets en el repositorio
- Usar siempre HTTPS para comunicación externa
- Implementar rate limiting en Nginx
- Regularmente rotar passwords y SSH keys
- Mantener Docker images actualizadas

### Performance
- Monitorear uso de CPU/RAM en Ollama
- Configurar adecuadamente volúmenes persistentes
- Implementar logging estructurado
- Considerar cluster para alta disponibilidad

### Escalabilidad
- Diseñar APIs para escalabilidad horizontal
- Implementar caché apropiado
- Considerar particionamiento de datos en ArangoDB
- Planificar estrategia de backup/restore

---

## 📞 Soporte y Contacto

- **Admin Repo GitHub**: [Nombre del administrador]
- **Soporte Servidor**: [Equipo de infraestructura]
- **Documentación actualizada**: https://github.com/empresa/empresa-infra/docs

**Última actualización**: Febrero 2026
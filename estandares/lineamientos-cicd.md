# Lineamientos de CI/CD

[Volver al indice](/)

---

## 1. Objetivo

Definir el pipeline estandar de integracion y despliegue continuo para los microservicios de la plataforma VIP, desde el commit hasta la ejecucion en produccion.

---

## 2. Pipeline General

```
  Commit ──▶ Build ──▶ Test ──▶ Scan ──▶ Publish ──▶ Deploy
    │          │         │        │         │           │
    ▼          ▼         ▼        ▼         ▼           ▼
 Pre-commit  Maven    Unit +   Seguridad  ECR/ACR    K8s Apply
 hooks       compile  Integration          Image      (por ambiente)
```

---

## 3. Pre-commit Hooks

Todo repositorio de microservicio debe tener hooks de pre-commit configurados:

| Hook | Herramienta | Proposito |
|------|-------------|-----------|
| Formato de codigo | `spotless` (Java) | Garantizar estilo consistente |
| Deteccion de secretos | `trufflehog` o `gitleaks` | Evitar que passwords/tokens se comitan |
| Linting | `checkstyle` | Verificar reglas de estilo Java |
| Commit message | `commitlint` | Formato: `tipo(alcance): descripcion` |

### Formato de Commit Messages

```
tipo(alcance): descripcion corta

feat(catastro): agregar endpoint de consulta por NPN
fix(errores): corregir mapeo de error INT-002 en adaptador SNR
docs(canonical): actualizar campos del request
refactor(cache): extraer logica de invalidacion a servicio
```

Tipos permitidos: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `perf`.

---

## 4. Etapa: Build

| Paso | Herramienta | Detalle |
|------|-------------|---------|
| Compilacion | Maven 3 + Java 17 | `mvn clean compile` |
| Empaquetado | Maven | `mvn package -DskipTests` |
| Imagen Docker | Docker / Buildah | Multi-stage build |

### Dockerfile Estandar

```dockerfile
FROM eclipse-temurin:17-jre-alpine AS runtime
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s \
  CMD wget -qO- http://localhost:8080/actuator/health/liveness || exit 1
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Reglas:**
- Usar imagenes base **alpine** para reducir superficie de ataque.
- No instalar herramientas de debug en la imagen final (curl, bash, etc.).
- No copiar archivos `.env`, `application-local.yml` ni secretos a la imagen.

---

## 5. Etapa: Test

| Tipo | Framework | Cobertura Minima |
|------|-----------|:----------------:|
| Unitarios | JUnit 5 + Mockito | 80% |
| Integracion | Spring Boot Test + Testcontainers | Endpoints criticos |
| Contrato | Spring Cloud Contract (opcional) | Contratos del Canonical |

**Reglas:**
- Los tests de integracion deben usar **Testcontainers** con PostgreSQL y Redis reales, no mocks de base de datos.
- Todo endpoint del Canonical debe tener al menos un test de integracion que valide la estructura de request/response.
- Los tests deben ejecutarse en el pipeline. Si fallan, el pipeline se detiene.

---

## 6. Etapa: Scan de Seguridad

| Herramienta | Que analiza | Cuando falla |
|-------------|-------------|:------------:|
| **Trivy** | Vulnerabilidades en imagen Docker y dependencias | CVE critico o alto sin parche |
| **OWASP Dependency-Check** | Vulnerabilidades en dependencias Maven | CVE con CVSS >= 7.0 |
| **SonarQube** (opcional) | Codigo fuente, bugs, code smells | Quality gate no cumplido |
| **trufflehog** | Secretos en el repositorio | Cualquier secreto detectado |

**Reglas:**
- El scan de Trivy es **obligatorio** antes de publicar la imagen.
- Las vulnerabilidades criticas **bloquean** el deploy a QA y produccion.
- Las vulnerabilidades altas deben tener un plan de remediacion dentro de 7 dias.

---

## 7. Etapa: Publish

### Imagen Docker

```bash
# Tag con semver + commit SHA corto
docker tag app registry.vortexbird.com/vip/acm-catastro-service-consulta:1.2.3
docker tag app registry.vortexbird.com/vip/acm-catastro-service-consulta:1.2.3-abc1234

# Push
docker push registry.vortexbird.com/vip/acm-catastro-service-consulta:1.2.3
```

| Ambiente | Registry | Tag Mutability |
|----------|----------|:--------------:|
| Dev | ECR/ACR | Mutable |
| QA | ECR/ACR | **Inmutable** |
| Prod | ECR/ACR | **Inmutable** |

### Lifecycle Policy (ECR)

- Retener las ultimas **30 imagenes tagueadas**.
- Eliminar imagenes sin tag despues de **7 dias**.

---

## 8. Etapa: Deploy

### Estrategia por Ambiente

| Ambiente | Estrategia | Aprobacion |
|----------|-----------|:----------:|
| Dev | Automatico en merge a `develop` | No |
| QA | Automatico en merge a `release/*` | No |
| Prod | Manual, requiere aprobacion | Si (lider tecnico) |

### Despliegue en Kubernetes

```bash
# Actualizar imagen en el deployment
kubectl set image deployment/acm-catastro-service-consulta \
  acm-catastro-service-consulta=registry.vortexbird.com/vip/acm-catastro-service-consulta:1.2.3 \
  -n acm-catastro

# Verificar rollout
kubectl rollout status deployment/acm-catastro-service-consulta -n acm-catastro
```

### Rollback

Si el readiness probe falla despues del deploy:

```bash
kubectl rollout undo deployment/acm-catastro-service-consulta -n acm-catastro
```

---

## 9. Branching Model

```
main (produccion)
  └── release/1.2.0 (QA → produccion)
        └── develop (integracion)
              ├── feature/CTR-001-consulta-predio
              ├── feature/CTR-002-avaluo-catastral
              └── fix/VLD-003-longitud-campo
```

| Branch | Proposito | Deploy a |
|--------|-----------|----------|
| `main` | Produccion estable | Prod (manual) |
| `release/*` | Candidato a produccion | QA (automatico) |
| `develop` | Integracion de features | Dev (automatico) |
| `feature/*` | Feature individual | — (solo CI) |
| `fix/*` | Bug fix | — (solo CI) |
| `hotfix/*` | Fix urgente en produccion | Prod (manual, expedito) |

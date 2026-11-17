# Práctica Calificada: Persistencia y Comunicación Multi-componente
## Kubernetes Cloud + Seguridad Profesional (S11 / AP8)

Este repositorio contiene la arquitectura completa, el código fuente y los manifiestos de Kubernetes para desplegar una aplicación multi-componente resiliente, segura y altamente disponible.

---

## 🏛️ Arquitectura del Sistema

El sistema implementa tres capas principales dentro de un espacio de nombres aislado (`practica-s11`):

1. **Capa 1: Persistencia NoSQL (MongoDB Stack)**
   - Base de datos NoSQL desplegada como `Deployment` conectada a un `PersistentVolumeClaim` (`mongo-pvc`) de 1Gi montado en `/data/db`.
   - Garantiza la persistencia de datos ante fallos o reinicio de Pods.
   - Credenciales administrativas seguras inyectadas desde un `Secret` (`db-secret`).
   - Exposición interna exclusiva vía `Service` de tipo `ClusterIP` en el puerto `27017` (`mongo-service`).

2. **Capa 2: Procesamiento Reactivo (Spring WebFlux Backend Stack)**
   - Microservicio reactivo no bloqueante desarrollado en **Java 17** con **Spring WebFlux**, **Spring Data Reactive MongoDB** y **Spring Boot Actuator**.
   - Externalización de configuración no sensible mediante `ConfigMap` (`app-config`) y credenciales confidenciales mediante `Secret` (`db-secret`), inyectadas simultáneamente vía `envFrom`.
   - Sondas de salud (`livenessProbe` y `readinessProbe`) apuntando al endpoint de Actuator `/actuator/health`.
   - Exposición externa para clientes mediante `Service` de tipo `NodePort` en el puerto `30080`.

3. **Capa 3: Comunicación e Integración Multi-componente**
   - Resolución de nombres DNS interna mediante **CoreDNS** (`mongo-service:27017`).
   - Conectividad reactiva extremo a extremo para operaciones CRUD.

---

## 📁 Estructura del Repositorio

Cumpliendo estrictamente las especificaciones de la guía:

```plaintext
/
├── k8s/
│   ├── 00-namespace.yaml        # Aislamiento del entorno (Namespace practica-s11)
│   ├── 01-mongo-pvc.yaml         # Almacenamiento persistente (PVC ReadWriteOnce 1Gi)
│   ├── 02-app-configmap.yaml     # Configuración no sensible (DB_HOST, DB_PORT, DB_NAME, APP_PORT)
│   ├── 03-db-secret.yaml         # Credenciales de BD codificadas en Base64
│   ├── 04-mongo-stack.yaml       # Deployment de MongoDB y Service ClusterIP
│   └── 05-backend-stack.yaml     # Deployment de Spring WebFlux y Service NodePort
├── src/                          # Código fuente de la aplicación Spring WebFlux
├── pom.xml                       # Dependencias y construcción con Maven
├── Dockerfile                    # Definición de la imagen del contenedor
└── README.md                     # Documentación y guía de arquitectura
```

---

## ⚙️ Pasos de Despliegue en el Clúster

### 1. Construir la imagen del Backend (si no está disponible en el registro)
```bash
docker build -t backend-webflux:latest .
```

### 2. Aplicar los manifiestos en orden secuencial
```bash
# Paso 1: Crear el Namespace
kubectl apply -f k8s/00-namespace.yaml

# Paso 2: Crear el PersistentVolumeClaim
kubectl apply -f k8s/01-mongo-pvc.yaml

# Paso 3: Crear el ConfigMap de la aplicación
kubectl apply -f k8s/02-app-configmap.yaml

# Paso 4: Crear el Secret de credenciales
kubectl apply -f k8s/03-db-secret.yaml

# Paso 5: Desplegar el Stack de MongoDB (Deployment + Service ClusterIP)
kubectl apply -f k8s/04-mongo-stack.yaml

# Paso 6: Desplegar el Stack del Backend (Deployment + Service NodePort)
kubectl apply -f k8s/05-backend-stack.yaml
```

## 🔒 Detalles de Seguridad y Configuración
- **Contraseñas**: El Secret almacena las credenciales codificadas en Base64:
  - `MONGO_INITDB_ROOT_USERNAME`: `admin` (`YWRtaW4=`)
  - `MONGO_INITDB_ROOT_PASSWORD`: `SecretPassword123!` (`U2VjcmV0UGFzc3dvcmQxMjMh`)
- **Aislamiento**: MongoDB está protegido con un Service `ClusterIP` que solo acepta conexiones internas dentro del clúster de Kubernetes.

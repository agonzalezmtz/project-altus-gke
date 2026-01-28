# Project Altus: GKE Global Batch Engine

**Project Altus** is a reference implementation of the **"Divide" architectural strategy** for Google Kubernetes Engine (GKE). [cite_start]It demonstrates how to orchestrate a distributed batch processing engine capable of executing **10,000 concurrent risk calculations** (financial simulations) in under **20 minutes**[cite: 200].

## 📋 Executive Summary

Legacy cluster architectures often hit physical limits when scaling burst workloads. Project Altus validates that by decoupling the "Control Plane" from the "Compute Plane" using GKE Enterprise, organizations can achieve:
* [cite_start]**Massive Scale:** 10,000 concurrent pods without saturating the Kubernetes API Server[cite: 203, 204].
* [cite_start]**70% Cost Reduction:** By leveraging Spot VMs in secondary regions for the heavy lifting.
* [cite_start]**Global Reach:** Bypassing regional IP quota limits and network bottlenecks[cite: 205].

## 🏗 Technical Architecture: Hub-and-Spoke Fleet

[cite_start]To overcome physical barriers like IP exhaustion and the "Thundering Herd" effect on the API Server, the infrastructure is divided into a **3-cluster GKE Fleet**[cite: 206].

![Architecture Diagram](https://via.placeholder.com/800x400?text=Architecture+Diagram+Placeholder)

| Cluster Role | Region | Type | Description |
| :--- | :--- | :--- | :--- |
| **Cluster-Hub** | `us-east1` | **GKE Autopilot** | Acts as the **"Brain"**. Hosts the Job Queue (Redis), Results Dashboard, and Observability stack. [cite_start]Stable control plane[cite: 207]. |
| **Cluster-Worker-A** | `us-central1` | **GKE Standard** | Acts as **"Muscle"**. [cite_start]Disposable cluster using **Spot VMs** for heavy compute[cite: 209]. |
| **Cluster-Worker-B** | `europe-west1` | **GKE Standard** | Acts as **"Muscle"**. [cite_start]Disposable cluster using **Spot VMs** for heavy compute[cite: 209]. |

### Key Technologies

* [cite_start]**Cloud Service Mesh:** Enables the 10,000 worker pods to connect securely to the Hub's Job Queue using internal DNS (`queue.prod.svc.cluster.local`) across regions without complex VPN tunnels[cite: 212].
* [cite_start]**Config Sync:** Pushes Kubernetes Job manifests to all worker clusters simultaneously, ensuring unified governance and drift detection.
* [cite_start]**Spot VMs:** Exclusively used in worker clusters to handle the burst workload, optimized with Node Termination Handlers[cite: 209, 275].
* [cite_start]**GKE Image Streaming:** Ensures nodes boot applications in under 4 seconds by streaming container data, avoiding network NAT saturation during the burst [cite: 287-289].

## 📂 Repository Structure

[cite_start]This repository follows the **Hierarchical Inheritance** pattern for Fleet management to manage complexity at scale[cite: 251]:

```text
├── system/               # Global policies (Security Agents, Logging) applied to ALL clusters [cite: 252]
├── fleets/
│   ├── hub/              # Configs for the Control Plane (Redis, Dashboards)
│   └── workers/          # Configs for Compute Planes (Spot VMs, Job Runners) [cite: 253]
├── namespaces/           # Namespace quotas and RBAC
└── workloads/            # The Batch Job Manifests
```

⚙️ Execution Flow

Fleet Connectivity: Cloud Service Mesh is enabled to create a unified trust domain, issuing mTLS certificates to every workload via Workload ID.


The Burst: A simple K8s Job manifest is pushed via Config Sync to the repository.


The Scale: 5,000 pods spin up in us-central1 and 5,000 in europe-west1 instantly, processing the queue in parallel.

🚀 Outcomes
Control Plane Isolation: The primary "Hub" cluster remained stable. The massive scheduling load of 10,000 pods was offloaded to the disposable "Worker" control planes.


Cost Efficiency: Shifting the burst workload to Spot VMs in cheaper regions resulted in a 70% cost reduction compared to running on reserved instances.


Resilience: The system successfully avoided the "Noisy Neighbor" effect and IP exhaustion common in single-cluster setups.

Project Altus is a reference implementation for "Navigating Large-Scale GKE".


Steps to create infra:

# 1. Definir variables globales
export PROJECT_ID=$(gcloud config get-value project)
export FLEET_HOST_PROJECT=$PROJECT_ID

# 2. Habilitar APIs necesarias (GKE, Fleet, Mesh, MCS)
gcloud services enable \
    container.googleapis.com \
    gkehub.googleapis.com \
    multiclusterservicediscovery.googleapis.com \
    trafficdirector.googleapis.com \
    anthos.googleapis.com \
    --project=$PROJECT_ID

gcloud container fleet multi-cluster-services enable

    # 1. Crear la VPC en modo custom
export NETWORK_NAME="vpc-altus"
gcloud compute networks create $NETWORK_NAME --subnet-mode=custom

# 2. Crear las 3 subredes (una para cada clúster)
# Subred para el Hub (us-east1)
gcloud compute networks subnets create sb-hub-east1 \
    --network $NETWORK_NAME \
    --region us-east1 \
    --range 10.0.1.0/24

# Subred para Worker A (us-central1)
gcloud compute networks subnets create sb-worker-a-central1 \
    --network $NETWORK_NAME \
    --region us-central1 \
    --range 10.0.2.0/24

# Subred para Worker B (europe-west1)
gcloud compute networks subnets create sb-worker-b-europe1 \
    --network $NETWORK_NAME \
    --region europe-west1 \
    --range 10.0.3.0/24

######## Aseguramos solo el ID del proyecto
export PROJECT_ID=$(gcloud config get-value project)

echo "--------------------------------------------------------"
echo "🔧 CORRIGIENDO INFRAESTRUCTURA DE RED (Red: vpc-altus)"
echo "--------------------------------------------------------"

# ==========================================
# 1. HUB (us-east1) - El Cerebro
# ==========================================
echo "1️⃣ Configurando Hub (us-east1)..."

# Crear Router (Si ya existe, dará error, es normal. Ignorar.)
gcloud compute routers create router-east1 \
    --project=$PROJECT_ID \
    --region us-east1 \
    --network vpc-altus \
    --quiet

# Crear NAT
gcloud compute routers nats create nat-east1 \
    --project=$PROJECT_ID \
    --router router-east1 \
    --region us-east1 \
    --auto-allocate-nat-external-ips \
    --nat-all-subnet-ip-ranges \
    --quiet

# ACTIVAR ACCESO PRIVADO (Nombre correcto: sb-hub-east1)
gcloud compute networks subnets update sb-hub-east1 \
    --region us-east1 \
    --project=$PROJECT_ID \
    --enable-private-ip-google-access


# ==========================================
# 2. WORKER A (us-central1) - Músculo USA
# ==========================================
echo "2️⃣ Configurando Worker A (us-central1)..."

# Crear Router
gcloud compute routers create router-central1 \
    --project=$PROJECT_ID \
    --region us-central1 \
    --network vpc-altus \
    --quiet

# Crear NAT
gcloud compute routers nats create nat-central1 \
    --project=$PROJECT_ID \
    --router router-central1 \
    --region us-central1 \
    --auto-allocate-nat-external-ips \
    --nat-all-subnet-ip-ranges \
    --quiet

# ACTIVAR ACCESO PRIVADO (CORREGIDO: sb-worker-a-central1)
gcloud compute networks subnets update sb-worker-a-central1 \
    --region us-central1 \
    --project=$PROJECT_ID \
    --enable-private-ip-google-access


# ==========================================
# 3. WORKER B (europe-west1) - Músculo Europa
# ==========================================
echo "3️⃣ Configurando Worker B (europe-west1)..."

# Crear Router
gcloud compute routers create router-europe1 \
    --project=$PROJECT_ID \
    --region europe-west1 \
    --network vpc-altus \
    --quiet

# Crear NAT
gcloud compute routers nats create nat-europe1 \
    --project=$PROJECT_ID \
    --router router-europe1 \
    --region europe-west1 \
    --auto-allocate-nat-external-ips \
    --nat-all-subnet-ip-ranges \
    --quiet

# ACTIVAR ACCESO PRIVADO (CORREGIDO: sb-worker-b-europe1)
gcloud compute networks subnets update sb-worker-b-europe1 \
    --region europe-west1 \
    --project=$PROJECT_ID \
    --enable-private-ip-google-access

echo "--------------------------------------------------------"
echo "✅ INFRAESTRUCTURA DE RED COMPLETADA (NOMBRES CORREGIDOS)"
echo "--------------------------------------------------------"

  # Crear clusters hub y esclavos

gcloud container clusters create-auto cluster-hub \
    --region us-east1 \
    --project=$PROJECT_ID \
    --release-channel "regular" \
    --network $NETWORK_NAME \
    --subnetwork sb-hub-east1 \
    --cluster-ipv4-cidr "10.100.0.0/20" \
    --services-ipv4-cidr "10.101.0.0/20" \
    --enable-private-nodes \
    --master-ipv4-cidr "172.16.0.0/28"

gcloud container clusters create cluster-worker-a \
    --region us-central1 \
    --project=$PROJECT_ID \
    --release-channel "regular" \
    --machine-type "e2-standard-4" \
    --spot \
    --enable-image-streaming \
    --workload-pool "${PROJECT_ID}.svc.id.goog" \
    --network $NETWORK_NAME \
    --subnetwork sb-worker-a-central1 \
    --cluster-ipv4-cidr "10.102.0.0/20" \
    --services-ipv4-cidr "10.103.0.0/20" \
    --enable-ip-alias \
    --enable-private-nodes \
    --master-ipv4-cidr "172.16.0.16/28" \
    --shielded-secure-boot \
    --shielded-integrity-monitoring

gcloud container clusters create cluster-worker-b \
    --region europe-west1 \
    --project=$PROJECT_ID \
    --release-channel "regular" \
    --machine-type "e2-standard-4" \
    --spot \
    --enable-image-streaming \
    --workload-pool "${PROJECT_ID}.svc.id.goog" \
    --network $NETWORK_NAME \
    --subnetwork sb-worker-b-europe1 \
    --cluster-ipv4-cidr "10.104.0.0/20" \
    --services-ipv4-cidr "10.105.0.0/20" \
    --enable-ip-alias \
    --enable-private-nodes \
    --master-ipv4-cidr "172.16.0.32/28" \
    --shielded-secure-boot \
    --shielded-integrity-monitoring


  # 1. Registrar el Hub (Cerebro)
gcloud container fleet memberships register cluster-hub \
    --gke-cluster us-east1/cluster-hub \
    --enable-workload-identity \
    --project $PROJECT_ID

# 2. Registrar Worker A (Músculo 1)
gcloud container fleet memberships register cluster-worker-a \
    --gke-cluster us-central1/cluster-worker-a \
    --enable-workload-identity \
    --project $PROJECT_ID

# 3. Registrar Worker B (Músculo 2)
gcloud container fleet memberships register cluster-worker-b \
    --gke-cluster europe-west1/cluster-worker-b \
    --enable-workload-identity \
    --project $PROJECT_ID

  # Paso 1: Habilitar la Malla en la Flota
Ejecuta esto para encender el servicio de "Service Mesh" en tu proyecto:

gcloud container fleet mesh enable --project $PROJECT_ID

# Paso 2: Activar la Gestión Automática (Managed Mesh)
En lugar de instalar Istio manualmente (que es doloroso), le diremos a Google: "Gestiona tú el plano de control de la malla en mis 3 clústeres".

# Configurar Hub (us-east1)
gcloud container fleet mesh update \
    --management automatic \
    --memberships cluster-hub \
    --project $PROJECT_ID

# Configurar Worker A (us-central1)
gcloud container fleet mesh update \
    --management automatic \
    --memberships cluster-worker-a \
    --project $PROJECT_ID

# Configurar Worker B (europe-west1)
gcloud container fleet mesh update \
    --management automatic \
    --memberships cluster-worker-b \
    --project $PROJECT_ID


# Paso 3: Activar la "Inyección de Sidecars"
Para que la magia funcione, cada Pod que creemos debe tener un acompañante (el proxy Envoy) que maneje el tráfico. Activamos esto poniendo una etiqueta en el namespace default de los 3 clústeres.

# 1. Obtener tu IP pública actual
export SHELL_IP=$(curl -s https://api.ipify.org)
echo "Autorizando la IP: $SHELL_IP"

# 2. Autorizar Cluster-Worker-A (US Central)
echo "Abriendo firewall para Worker A..."
gcloud container clusters update cluster-worker-a \
    --region us-central1 \
    --enable-master-authorized-networks \
    --master-authorized-networks $SHELL_IP/32 \
    --project $PROJECT_ID

# 3. Autorizar Cluster-Worker-B (Europa)
echo "Abriendo firewall para Worker B..."
gcloud container clusters update cluster-worker-b \
    --region europe-west1 \
    --enable-master-authorized-networks \
    --master-authorized-networks $SHELL_IP/32 \
    --project $PROJECT_ID

# Array de clústeres y regiones
declare -A clusters
clusters=( ["cluster-hub"]="us-east1" ["cluster-worker-a"]="us-central1" ["cluster-worker-b"]="europe-west1" )

# Bucle para etiquetar el namespace 'default' en todos
for cluster in "${!clusters[@]}"; do
    REGION=${clusters[$cluster]}
    echo "Configurando $cluster en $REGION..."
    
    # Obtener credenciales
    gcloud container clusters get-credentials $cluster --region $REGION --project $PROJECT_ID
    
    # Etiquetar namespace para inyección automática de Istio/Mesh
    kubectl label namespace default istio-injection=enabled --overwrite
    
    echo "---------------------------------------------------"
done



# ##################PASO 1: Encender el Cerebro (Redis en el Hub)
Objetivo: Instalar la base de datos y hacerla visible para EE.UU. y Europa.

Copia y pega todo este bloque en tu terminal:

Bash

# 1. Asegurar variables
export PROJECT_ID=$(gcloud config get-value project)

# 2. Conectarse al Hub (us-east1)
echo "Conectando al Hub..."
gcloud container clusters get-credentials cluster-hub --region us-east1 --project $PROJECT_ID

# 3. Instalar Redis
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  labels:
    app: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:alpine
        ports:
        - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
EOF

# 4. Exportar Redis a la Flota (MCS)
# Esto crea el DNS mágico: redis.default.svc.clusterset.local
kubectl apply -f - <<EOF
apiVersion: net.gke.io/v1
kind: ServiceExport
metadata:
  name: redis
  namespace: default
EOF

echo "✅ Redis desplegado y exportado."

# ###########  Paso 1: Desplegar el Ejército Dormido (Workers)  ################
Vamos a enviar la configuración a us-central1 y europe-west1. Fíjate que la variable REDIS_HOST apunta al DNS Global de la Flota (.clusterset.local).

Copia y pega este bloque completo:


# Definir el manifiesto del Worker
cat <<EOF > worker-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: altus-worker
  labels:
    app: altus-worker
spec:
  # Empezamos dormidos (0 pods) para dar la orden manual
  replicas: 0
  selector:
    matchLabels:
      app: altus-worker
  template:
    metadata:
      labels:
        app: altus-worker
    spec:
      containers:
      - name: worker
        image: gcr.io/$PROJECT_ID/altus-worker:v1
        env:
        - name: REDIS_HOST
          value: "redis.default.svc.clusterset.local" # <--- LA MAGIA DE MCS
        resources:
          requests:
            cpu: "100m"
            memory: "100Mi"
EOF

# Desplegar en Worker A (US Central)
echo "Desplegando en Worker A..."
gcloud container clusters get-credentials cluster-worker-a --region us-central1 --project $PROJECT_ID
kubectl apply -f worker-deployment.yaml

# Desplegar en Worker B (Europa)
echo "Desplegando en Worker B..."
gcloud container clusters get-credentials cluster-worker-b --region europe-west1 --project $PROJECT_ID
kubectl apply -f worker-deployment.yaml

Paso 2: Cargar la Munición (Generar 10k Jobs)
Volvemos al Hub para llenar la base de datos. Usaremos un pequeño script para inyectar 10,000 claves en la lista jobs.


# 1. Volver al Hub
gcloud container clusters get-credentials cluster-hub --region us-east1 --project $PROJECT_ID

# 2. Obtener el nombre del pod de Redis
REDIS_POD=$(kubectl get pod -l app=redis -o jsonpath="{.items[0].metadata.name}")

# 3. Inyectar 10,000 trabajos (Esto tomará unos 10-15 segundos)
echo "Inyectando 10,000 trabajos en $REDIS_POD..."
kubectl exec -it $REDIS_POD -- /bin/sh -c "
  for i in \$(seq 1 10000); do
    redis-cli rpush jobs \"job-\$i\" > /dev/null
    if [ \$((i % 1000)) -eq 0 ]; then echo \"Generados \$i trabajos...\"; fi
  done
"

1. Instalar la Web App en el Hub 🖥️
Bash

# 1. Asegurar variables y conectar al Hub
export PROJECT_ID=$(gcloud config get-value project)
echo "🔌 Conectando al Hub (us-east1)..."
gcloud container clusters get-credentials cluster-hub --region us-east1 --project $PROJECT_ID

# 2. Desplegar Redis Commander y crear su IP pública
echo "📦 Instalando interfaz gráfica..."
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-web
  labels:
    app: redis-web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis-web
  template:
    metadata:
      labels:
        app: redis-web
    spec:
      containers:
      - name: redis-web
        image: rediscommander/redis-commander:latest
        env:
        - name: REDIS_HOSTS
          value: "local:redis:6379"
        ports:
        - containerPort: 8081
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: redis-web-lb
spec:
  type: LoadBalancer
  selector:
    app: redis-web
  ports:
  - port: 80
    targetPort: 8081
EOF
2. Obtener la dirección IP para entrar 🌐
Ejecuta este comando y espera (aprox. 1 minuto) hasta que debajo de EXTERNAL-IP aparezca una dirección numérica (ej. 34.123.45.67):

Bash

kubectl get service redis-web-lb --watch


# 1. Conectar al Hub
export PROJECT_ID=$(gcloud config get-value project)
gcloud container clusters get-credentials cluster-hub --region us-east1 --project $PROJECT_ID

# 2. Buscar el Pod de Redis
REDIS_POD=$(kubectl get pod -l app=redis -o jsonpath="{.items[0].metadata.name}")

# 3. ¡Inyectar datos!
echo "🔥 Inyectando 10,000 trabajos en $REDIS_POD..."
kubectl exec -it $REDIS_POD -- /bin/sh -c "
  for i in \$(seq 1 10000); do
    redis-cli rpush jobs \"job-\$i\" > /dev/null
    if [ \$((i % 2000)) -eq 0 ]; then echo \"Generados \$i trabajos...\"; fi
  done
"


echo "🔥 ¡SOLTANDO AL KRAKEN GLOBAL!"

# 1. Despertar a EE.UU. (Worker A)
echo "🇺🇸 Activando 50 servidores en USA..."
gcloud container clusters get-credentials cluster-worker-a --region us-central1 --project $PROJECT_ID
kubectl scale deployment altus-worker --replicas=50

# 2. Despertar a Europa (Worker B)
echo "🇪🇺 Activando 50 servidores en Europa..."
gcloud container clusters get-credentials cluster-worker-b --region europe-west1 --project $PROJECT_ID
kubectl scale deployment altus-worker --replicas=50

echo "✅ ¡Ataque iniciado! CORRE A LA WEB DE REDIS."


cat <<EOF > worker-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: altus-worker
  labels:
    app: altus-worker
spec:
  replicas: 0 # Lo ponemos en 0 un momento para reiniciar la estrategia
  selector:
    matchLabels:
      app: altus-worker
  template:
    metadata:
      labels:
        app: altus-worker
    spec:
      containers:
      - name: worker
        image: gcr.io/$PROJECT_ID/altus-worker:v1
        env:
        - name: REDIS_HOST
          value: "redis.default.svc.clusterset.local"
        resources:
          requests:
            cpu: "10m"      # <--- 0.01 CPU (Diminuto)
            memory: "32Mi"  # <--- 32 MB RAM
EOF

export PROJECT_ID=$(gcloud config get-value project)

echo "📉 APLICANDO REDUCCIÓN DE TAMAÑO..."

# Actualizar Worker A (USA)
echo "🇺🇸 Actualizando Worker A..."
gcloud container clusters get-credentials cluster-worker-a --region us-central1 --project $PROJECT_ID
kubectl apply -f worker-deployment.yaml

# Actualizar Worker B (Europa)
echo "🇪🇺 Actualizando Worker B..."
gcloud container clusters get-credentials cluster-worker-b --region europe-west1 --project $PROJECT_ID
kubectl apply -f worker-deployment.yaml

echo "✅ Los pods ahora son 10 veces más ligeros."

ok

# 1. CAPTURAR PROYECTO (¡Crucial!)
export PROJECT_ID=$(gcloud config get-value project)
echo "✅ Proyecto detectado: $PROJECT_ID"

# 2. BORRAR LO ROTO (Limpieza profunda)
echo "🧹 Borrando despliegue corrupto..."
gcloud container clusters get-credentials cluster-worker-a --region us-central1 --project $PROJECT_ID
kubectl delete deployment altus-worker --ignore-not-found=true

gcloud container clusters get-credentials cluster-worker-b --region europe-west1 --project $PROJECT_ID
kubectl delete deployment altus-worker --ignore-not-found=true

# 3. CREAR ARCHIVO CORRECTO (Sin el error //)
echo "🔧 Generando worker-deployment.yaml corregido..."
cat <<EOF > worker-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: altus-worker
  labels:
    app: altus-worker
spec:
  replicas: 0  # Empezamos en 0 para seguridad
  selector:
    matchLabels:
      app: altus-worker
  template:
    metadata:
      labels:
        app: altus-worker
    spec:
      containers:
      - name: worker
        image: gcr.io/$PROJECT_ID/altus-worker:v1  # <--- AHORA SÍ LLEVARÁ EL ID
        env:
        - name: REDIS_HOST
          value: "redis.default.svc.clusterset.local"
        resources:
          requests:
            cpu: "10m"      # Nano-CPU (caben miles)
            memory: "32Mi"
EOF

# 4. APLICAR CORRECCIÓN
echo "🚀 Aplicando nuevo ejército..."
gcloud container clusters get-credentials cluster-worker-a --region us-central1 --project $PROJECT_ID
kubectl apply -f worker-deployment.yaml

gcloud container clusters get-credentials cluster-worker-b --region europe-west1 --project $PROJECT_ID
kubectl apply -f worker-deployment.yaml

echo "✅ ¡LISTO! El error de imagen debería haber desaparecido."


# 1. Asegurar el ID del proyecto
export PROJECT_ID=$(gcloud config get-value project)

echo "👷‍♂️ CREANDO CÓDIGO DEL TRABAJADOR..."

# 2. Crear el script de Python (El cerebro que come trabajos)
cat <<EOF > worker.py
import os
import redis
import time
import sys

# Conexión a Redis
redis_host = os.environ.get('REDIS_HOST', 'localhost')
r = redis.StrictRedis(host=redis_host, port=6379, db=0)

print(f"🔥 Worker arrancando. Conectando a {redis_host}...")

while True:
    try:
        # Intentar sacar un trabajo de la lista 'jobs'
        # blpop espera si la lista está vacía, lpop no espera
        job = r.lpop('jobs')
        
        if job:
            # Fingimos que procesamos algo (muy rápido)
            # print(f"Comiendo trabajo: {job}") 
            pass 
        else:
            # Si no hay nada, dormimos un poco para no saturar CPU a lo tonto
            time.sleep(1)
            
    except Exception as e:
        print(f"Error: {e}")
        time.sleep(5)
EOF

# 3. Crear el Dockerfile (La receta de cocina)
cat <<EOF > Dockerfile
FROM python:3.9-slim
RUN pip install redis
COPY worker.py .
CMD ["python", "-u", "worker.py"]
EOF

echo "🍳 COCINANDO LA IMAGEN (Esto tardará unos 2 minutos)..."
# 4. Enviar a construir a la nube
gcloud builds submit -t gcr.io/$PROJECT_ID/altus-worker:v1 .


export PROJECT_ID=$(gcloud config get-value project)
echo "🔌 HABILITANDO AUTOSCALER (Ahora sí de verdad)..."

# 1. Activar en USA
echo "🇺🇸 Activando en Worker A..."
gcloud container clusters update cluster-worker-a \
    --region us-central1 \
    --enable-autoscaling \
    --min-nodes 1 --max-nodes 50 \
    --project $PROJECT_ID

# 2. Activar en Europa
echo "🇪🇺 Activando en Worker B..."
gcloud container clusters update cluster-worker-b \
    --region europe-west1 \
    --enable-autoscaling \
    --min-nodes 1 --max-nodes 50 \
    --project $PROJECT_ID

echo "✅ ¡Autoscaler encendido! Ahora espera 2 minutos a que empiece la magia."

CHECAR EL TAMANO DE LAS REDES



    

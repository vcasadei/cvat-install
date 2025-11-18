
# CVAT AI Annotation Lab (Stable v2.10.0 - GPU Ready)

Esta é uma solução de implantação do CVAT (Computer Vision Annotation Tool) otimizada para estabilidade, colaboração multiusuário e aceleração por GPU.

## 🌟 Visão Geral e Recursos

-   **Versão Estável:** Utiliza CVAT v2.10.0, Traefik v2.9 e PostgreSQL 12 para máxima estabilidade e evitar conflitos de migração.
    
-   **Acesso Único:** CVAT exposto na porta **8081** (para não conflitar com serviços padrão como JupyterHub).
    
-   **Shared Workspace:** Pasta de dados persistente e compartilhada (`~/cvat_share`) para importar datasets sem upload.
    
-   **AI Acelerada:** Suporte a modelos como **SAM** e **YOLO** rodando diretamente na GPU (via Nuclio).
    
-   **Correção Crítica de API:** Contorna o erro de incompatibilidade de versão do Docker API (1.24 vs 1.44).
    

----------

## 🛠️ PARTE I: Configuração do Servidor (Host)

Este passo é **obrigatório** para que os containers Nuclio e Traefik consigam se comunicar com o seu Docker Daemon.

### 1. Aplicar Correção de API do Docker

Você deve instruir o Docker Engine a aceitar clientes antigos (versão 1.24), resolvendo o erro `client version is too old`.

1.  Edite o arquivo de configuração do daemon (geralmente via `sudo nano /etc/docker/daemon.json`).
    
2.  **Adicione ou modifique** a linha `min-api-version`.
    

```json
{
  "min-api-version": "1.24"
}

```

3.  Reinicie o serviço do Docker para aplicar a mudança:
    
    
    ```bash
    sudo systemctl restart docker
    
    ```
    

### 2. Preparar Diretórios e Permissões


```bash
# Entrar na pasta do projeto
mkdir -p ~/cvat-gpu-lab
cd ~/cvat-gpu-lab

# Criar pastas de persistência e compartilhamento
mkdir -p ~/cvat_data/db ~/cvat_data/redis
mkdir -p ~/cvat_data/data ~/cvat_data/keys ~/cvat_data/logs 
mkdir -p ~/cvat_share

# Permissões (Usuário CVAT = ID 1000)
sudo chown -R 1000:1000 ~/cvat_data
sudo chown -R 1000:1000 ~/cvat_share

```

----------

## 💻 PARTE II: Deploy do CVAT

### 3. Clonar a Versão Estável


```bash
# Clona a versão v2.10.0 (Estável)
git clone -b v2.10.0 https://github.com/cvat-ai/cvat.git .

```

### 4. Criar Arquivos de Configuração

#### A. O Arquivo `.env` (Configuração Central)

Crie o arquivo `.env` na raiz:

```
# --- REDE ---
CVAT_HOST=172.25.6.20
CVAT_PORT=8081

# --- DADOS ---
CVAT_HOST_ROOT_DATA_DIR=/home/vc/cvat_data
CVAT_HOST_SHARE_DIR=/home/vc/cvat_share
CVAT_VERSION=v2.10.0

```

#### B. O Arquivo `docker-compose.override.yml`

Crie o arquivo de override com todas as configurações de portabilidade e GPU:


```yaml
version: '3.3'

services:
  cvat:
    image: cvat/server:v2.10.0
    environment:
      CVAT_SHARE_URL: "Mounted from ${CVAT_HOST_SHARE_DIR}"
      ALLOWED_HOSTS: '*'
    volumes:
      - ${CVAT_HOST_ROOT_DATA_DIR}/data:/home/django/data
      - ${CVAT_HOST_ROOT_DATA_DIR}/keys:/home/django/keys
      - ${CVAT_HOST_ROOT_DATA_DIR}/logs:/home/django/logs
      - ${CVAT_HOST_SHARE_DIR}:/home/django/share:ro

  cvat_db:
    image: postgres:12-alpine
    environment:
      POSTGRES_HOST_AUTH_METHOD: trust # Permite conexão do CVAT Server
    volumes:
      - ${CVAT_HOST_ROOT_DATA_DIR}/db:/var/lib/postgresql/data
  
  cvat_redis:
    image: redis:6-alpine
    volumes:
      - ${CVAT_HOST_ROOT_DATA_DIR}/redis:/data

  traefik:
    image: traefik:v2.9
    ports:
      - "${CVAT_PORT}:8080"
    environment:
      - DOCKER_API_VERSION=1.44 # Necessário para o cliente Go (Redundante, mas seguro)
    command:
      - "--entrypoints.web.address=:8080"
      - "--providers.docker.exposedbydefault=false"

  nuclio:
    image: quay.io/nuclio/dashboard:1.8.14-amd64
    environment:
      - DOCKER_API_VERSION=1.44
      - NUCLIO_TEMPLATES_GIT_URL= # Desativa busca de templates (Fix 500 Error)
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility

```

### 5. Patch Core File (Portas)

Comente as portas no arquivo original para evitar conflito com seu override:


```bash
sed -i 's/ports:/# ports:/g' docker-compose.yml
sed -i 's/- 8080:8080/# - 8080:8080/g' docker-compose.yml
sed -i 's/- 8090:8090/# - 8090:8090/g' docker-compose.yml

```

### 6. Executar o Sistema


```bash
export CVAT_HOST=172.25.6.20

docker compose \
  --env-file .env \
  -f docker-compose.yml \
  -f components/serverless/docker-compose.serverless.yml \
  -f docker-compose.override.yml \
  up -d --build

```

### 7. Pós-Instalação

1.  **Criar Admin:**
    
    
    ```bash
    docker exec -it cvat_server python3 manage.py createsuperuser
    ```

### 8. Instalar Modelos de IA (Final)


```bash
# Instalar nuctl (se necessário) e deployar SAM e YOLOv7...
# Baixa nuctl (Se ainda não tiver) wget 
https://github.com/nuclio/nuclio/releases/download/1.8.14/nuctl-1.8.14-linux-amd64 chmod +x nuctl-1.8.14-linux-amd64 sudo mv nuctl-1.8.14-linux-amd64 /usr/local/bin/nuctl 

# Cria projeto e deploya SAM e YOLO nuctl 
create project cvat --platform local

nuctl deploy --project-name cvat --path serverless/pytorch/facebookresearch/sam/nuclio --volume `pwd`/serverless/common:/opt/nuclio/common --platform local --resource-limit nvidia.com/gpu=1 --triggers '{"myHttpTrigger": {"maxWorkers": 1}}' 

nuctl deploy --project-name cvat \ --path serverless/onnx/WongKinYiu/yolov7/nuclio \ --platform local \ --resource-limit nvidia.com/gpu=1
```

Embora SAM (Segmentação) e YOLO (Detecção) resolvam a maioria dos problemas, é interessante instalar modelos que ofereçam funcionalidades complementares ou redundância para aumentar a precisão.

Com base na estrutura de modelos da sua versão do CVAT (`v2.49.0`), os modelos mais interessantes para complementar sua suíte de IA são:

#### 1. RetinaNet (Detecção de Objetos de Alta Qualidade)

Embora o YOLO seja rápido, o RetinaNet é um detector de objetos robusto e de alta precisão (R-CNN based). Instalá-lo oferece uma alternativa para quando o YOLO falhar ou para casos que exigem maior rigor.


```bash
nuctl deploy --project-name cvat \
  --path serverless/pytorch/facebookresearch/detectron2/retinanet_r101/nuclio \
  --platform local \
  --resource-limit nvidia.com/gpu=1

```

----------

#### 2. Mask R-CNN (Segmentação de Instância - TensorFlow)

Este modelo é essencial, pois ele faz duas coisas ao mesmo tempo: **Detecção** e **Segmentação** (criação de máscaras pixel a pixel). Além disso, ele é baseado em **TensorFlow**, o que garante que sua segunda stack de Deep Learning esteja funcional e pronta para uso.


```bash
nuctl deploy --project-name cvat \
  --path serverless/tensorflow/matterport/mask_rcnn/nuclio \
  --platform local \
  --resource-limit nvidia.com/gpu=1

```

----------

#### 3. SiamMask (Rastreamento de Objetos em Vídeo)

Este modelo oferece uma funcionalidade totalmente nova, crucial para anotação de vídeos: **Rastreamento**. Você marca o objeto no primeiro frame, e o SiamMask o rastreia automaticamente nos frames seguintes.


```bash
nuctl deploy --project-name cvat \
  --path serverless/pytorch/foolwood/siammask/nuclio \
  --platform local \
  --resource-limit nvidia.com/gpu=1

```


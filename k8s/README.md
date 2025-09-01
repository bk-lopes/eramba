# Eramba Kubernetes Manifests

Este diretório contém os manifestos Kubernetes para deploy do Eramba em um cluster Kubernetes (EKS ou MicroK8s).

## Arquivos incluídos

- `namespace.yaml` - Namespace para o Eramba
- `configmap.yaml` - Configurações gerais da aplicação
- `secret.yaml` - Senhas e dados sensíveis
- `persistent-volumes.yaml` - PVCs para dados persistentes
- `mysql.yaml` - Deployment e Service do MySQL
- `redis.yaml` - Deployment e Service do Redis
- `eramba.yaml` - Deployment e Service do Eramba
- `cron.yaml` - Deployment do Cron
- `apache-configs.yaml` - Configurações do Apache
- `crontab-config.yaml` - Configurações do Crontab
- `ingress.yaml` - Ingress com ALB Controller

## Pré-requisitos

1. Cluster Kubernetes (EKS ou MicroK8s)
2. ALB Ingress Controller instalado (para EKS)
3. StorageClass configurado (gp2 para EKS)

## Deploy

### Opção 1: Deploy automático (recomendado)

```bash
# Tornar o script executável
chmod +x k8s/deploy.sh

# Executar o deploy
./k8s/deploy.sh
```

### Opção 2: Deploy manual

### 1. Configurar variáveis

Edite os arquivos `configmap.yaml` e `secret.yaml` com suas configurações:

```bash
# Editar configurações gerais
vim k8s/configmap.yaml

# Editar senhas (em base64)
vim k8s/secret.yaml
```

### 2. Configurar certificados SSL

Edite o arquivo `apache-configs.yaml` e substitua os certificados SSL:

```yaml
data:
  mycert.crt: |
    -----BEGIN CERTIFICATE-----
    # Seu certificado aqui
    -----END CERTIFICATE-----
  mycert.key: |
    -----BEGIN PRIVATE KEY-----
    # Sua chave privada aqui
    -----END PRIVATE KEY-----
```

### 3. Configurar Ingress

Edite o arquivo `ingress.yaml`:

- Substitua `eramba.example.com` pelo seu domínio
- Substitua `arn:aws:acm:region:account:certificate/certificate-id` pelo ARN do seu certificado ACM

### 4. Aplicar os manifestos

```bash
# Aplicar em ordem
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/persistent-volumes.yaml
kubectl apply -f k8s/mysql.yaml
kubectl apply -f k8s/redis.yaml
kubectl apply -f k8s/apache-configs.yaml
kubectl apply -f k8s/crontab-config.yaml
kubectl apply -f k8s/eramba.yaml
kubectl apply -f k8s/cron.yaml
kubectl apply -f k8s/ingress.yaml
```

### 5. Verificar o deploy

```bash
# Verificar pods
kubectl get pods -n eramba

# Verificar services
kubectl get svc -n eramba

# Verificar ingress
kubectl get ingress -n eramba

# Logs do Eramba
kubectl logs -f deployment/eramba -n eramba
```

## Configurações importantes

### Volumes

Os volumes são configurados para evitar sobrescrever o conteúdo da imagem:

- `eramba-app-pvc`: Volume para a aplicação (/var/www/eramba)
- `eramba-data-pvc`: Volume para dados (/var/www/eramba/app/upgrade/data)
- `eramba-logs-pvc`: Volume para logs (/var/www/eramba/app/upgrade/logs)
- `mysql-data-pvc`: Volume para dados do MySQL

### Recursos

- MySQL: 512Mi-1Gi RAM, 250m-500m CPU
- Redis: 128Mi-256Mi RAM, 100m-200m CPU
- Eramba: 512Mi-1Gi RAM, 250m-500m CPU
- Cron: 256Mi-512Mi RAM, 100m-200m CPU

### Migração futura

Para migrar para RDS e ElastiCache:

1. Remover deployments do MySQL e Redis
2. Atualizar variáveis de ambiente no ConfigMap
3. Configurar conectividade de rede para RDS/ElastiCache

## Limpeza

Para remover todos os recursos do Eramba:

```bash
# Tornar o script executável
chmod +x k8s/cleanup.sh

# Executar a limpeza
./k8s/cleanup.sh
```

## Troubleshooting

### Problema: Arquivos não encontrados no Eramba

Se o Eramba apresentar erros como "Could not open input file: composer.phar" ou "rsync: change_dir failed":

**Solução implementada:** Removemos o volume que sobrescrevia o diretório `/var/www/eramba` da imagem. Agora apenas os diretórios de dados e logs são persistentes.

1. **Verificar logs do initContainer:**
   ```bash
   kubectl logs deployment/eramba -c init-volumes -n eramba
   ```

2. **Aplicar a correção:**
   ```bash
   kubectl apply -f k8s/eramba.yaml
   kubectl apply -f k8s/cron.yaml
   ```

3. **Verificar se os arquivos estão presentes:**
   ```bash
   kubectl exec -it deployment/eramba -n eramba -- ls -la /var/www/eramba/
   kubectl exec -it deployment/eramba -n eramba -- ls -la /var/www/eramba/app/upgrade/
   ```

### Problema: Pod do Eramba fica em Pending

Se o pod do Eramba ficar em status "Pending" com erro "nc: not found" no initContainer:

1. **Verificar logs do initContainer:**
   ```bash
   kubectl logs deployment/eramba -c init-eramba -n eramba
   ```

2. **Aplicar a correção:**
   ```bash
   kubectl apply -f k8s/eramba.yaml
   ```

3. **Verificar se o pod está funcionando:**
   ```bash
   kubectl get pods -n eramba
   ```

### Problema: MySQL não inicia

Se o MySQL não conseguir iniciar com erro de "Data Dictionary initialization failed":

1. **Verificar se o PVC foi criado corretamente:**
   ```bash
   kubectl get pvc -n eramba
   ```

2. **Deletar o PVC e recriar (CUIDADO: isso apagará os dados):**
   ```bash
   kubectl delete pvc mysql-data-pvc -n eramba
   kubectl apply -f persistent-volumes.yaml
   kubectl apply -f mysql.yaml
   ```

3. **Verificar logs do MySQL:**
   ```bash
   kubectl logs -f deployment/mysql -n eramba
   ```

### Verificar logs

```bash
# Logs do Eramba
kubectl logs -f deployment/eramba -n eramba

# Logs do MySQL
kubectl logs -f deployment/mysql -n eramba

# Logs do Cron
kubectl logs -f deployment/cron -n eramba
```

### Verificar conectividade

```bash
# Testar conectividade entre pods
kubectl exec -it deployment/eramba -n eramba -- nc -z mysql 3306
kubectl exec -it deployment/eramba -n eramba -- nc -z redis 6379
```

### Verificar volumes

```bash
# Verificar PVCs
kubectl get pvc -n eramba

# Verificar PVs
kubectl get pv
```

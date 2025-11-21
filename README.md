# argocd-gitops

Projeto de GitOps com ArgoCD, MySQL Operator e Argo Rollouts

## Arquitetura

Este projeto implementa:

- **ArgoCD**: Continuous Deployment com GitOps
- **MySQL Operator**: Gerenciamento de clusters MySQL
- **Argo Rollouts**: Deployments avançados com estratégias BlueGreen e Canary

## Estrutura do Projeto

```
.
├── apps/                           # Aplicações ArgoCD
│   ├── dev-app.yaml               # App dev (Helm)
│   ├── prod-app.yaml              # App prod (Helm)
│   ├── dev-rollout-app.yaml       # Rollout dev (BlueGreen)
│   └── prod-rollout-app.yaml      # Rollout prod (Canary)
├── operators/                      # Operadores
│   ├── mysql-operator.yaml        # MySQL Operator
│   └── argo-rollouts.yaml         # Argo Rollouts
├── rollout-dev.yaml               # Rollout BlueGreen para dev
├── rollout-prod.yaml              # Rollout Canary para prod
├── services-dev.yaml              # Services dev (active/preview)
├── services-prod.yaml             # Services prod (stable/canary)
├── mysql-dev.yaml                 # MySQL cluster dev
└── mysql-prod.yaml                # MySQL cluster prod
```

## Instalação

### 1. Instalar o Argo Rollouts

```bash
kubectl apply -f operators/argo-rollouts.yaml
```

Ou instalar o Argo Rollouts diretamente:

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

### 2. Instalar o kubectl plugin para Argo Rollouts

```bash
# Linux/Mac
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts

# Windows (PowerShell)
Invoke-WebRequest -Uri https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-windows-amd64 -OutFile kubectl-argo-rollouts.exe
Move-Item kubectl-argo-rollouts.exe C:\Windows\System32\
```

### 3. Aplicar os Rollouts

```bash
# Deploy do ambiente de desenvolvimento (BlueGreen)
kubectl apply -f apps/dev-rollout-app.yaml

# Deploy do ambiente de produção (Canary)
kubectl apply -f apps/prod-rollout-app.yaml
```

## Estratégias de Deploy

### BlueGreen (Desenvolvimento)

A estratégia BlueGreen mantém duas versões completas da aplicação:

- **Blue (Active)**: Versão atual em produção recebendo tráfego
- **Green (Preview)**: Nova versão para testes

**Características:**

- Auto-promoção **DESABILITADA** (`autoPromotionEnabled: false`)
- Requer promoção manual após validação
- Rollback instantâneo se houver problemas

#### Comandos para BlueGreen

```bash
# Visualizar status do rollout
kubectl argo rollouts get rollout ivodocker-dev -n dev

# Visualizar em tempo real
kubectl argo rollouts get rollout ivodocker-dev -n dev --watch

# Listar rollouts
kubectl argo rollouts list rollouts -n dev

# Promover manualmente a versão preview para active
kubectl argo rollouts promote ivodocker-dev -n dev

# Fazer rollback para versão anterior
kubectl argo rollouts undo ivodocker-dev -n dev

# Abortar um rollout em progresso
kubectl argo rollouts abort ivodocker-dev -n dev

# Reiniciar um rollout
kubectl argo rollouts restart ivodocker-dev -n dev

# Testar a versão preview
kubectl port-forward -n dev service/ivodocker-dev-preview 8080:80

# Acessar a versão active
kubectl port-forward -n dev service/ivodocker-dev-active 8081:80
```

#### Fluxo BlueGreen

1. Deploy de nova versão:

   ```bash
   # Atualizar a imagem no rollout-dev.yaml
   kubectl apply -f rollout-dev.yaml
   ```

2. Nova versão sobe no service `ivodocker-dev-preview`

3. Testar a versão preview:

   ```bash
   kubectl port-forward -n dev service/ivodocker-dev-preview 8080:80
   # Acessar http://localhost:8080
   ```

4. Se aprovado, promover manualmente:

   ```bash
   kubectl argo rollouts promote ivodocker-dev -n dev
   ```

5. Tráfego muda para a nova versão e antiga é desligada após 30s

### Canary (Produção)

A estratégia Canary incrementa gradualmente o tráfego para a nova versão:

- Começa com 20% do tráfego
- Aumenta para 40%, 60%, 80%
- Finaliza com 100%

**Características:**

- Rollout progressivo com pausas entre steps
- Primeira pausa é indefinida (manual)
- Demais pausas são de 30 segundos
- Permite análise e validação em cada etapa

#### Comandos para Canary

```bash
# Visualizar status do rollout
kubectl argo rollouts get rollout ivodocker-prod -n prod

# Visualizar em tempo real com gráfico
kubectl argo rollouts get rollout ivodocker-prod -n prod --watch

# Promover para próximo step (avançar de 20% para 40%, etc)
kubectl argo rollouts promote ivodocker-prod -n prod

# Pular todos os steps e ir direto para 100%
kubectl argo rollouts promote ivodocker-prod -n prod --full

# Fazer rollback
kubectl argo rollouts undo ivodocker-prod -n prod

# Abortar rollout e voltar para versão estável
kubectl argo rollouts abort ivodocker-prod -n prod

# Definir peso manualmente (ajustar tráfego)
kubectl argo rollouts set canary ivodocker-prod -n prod --weight 25

# Retomar rollout após pausa
kubectl argo rollouts promote ivodocker-prod -n prod

# Ver histórico de revisões
kubectl argo rollouts history ivodocker-prod -n prod

# Verificar análise de métricas (se configurado)
kubectl argo rollouts get rollout ivodocker-prod -n prod --analysis
```

#### Fluxo Canary

1. Deploy de nova versão:

   ```bash
   # Atualizar a imagem no rollout-prod.yaml
   kubectl apply -f rollout-prod.yaml
   ```

2. Rollout inicia com 20% do tráfego para canary

3. **Pausa indefinida** - Análise manual necessária:

   ```bash
   # Verificar logs, métricas, erros
   kubectl logs -n prod -l app=ivodocker,env=prod --tail=50

   # Se OK, promover para próximo step
   kubectl argo rollouts promote ivodocker-prod -n prod
   ```

4. Tráfego aumenta para 40% (pausa 30s)

5. Tráfego aumenta para 60% (pausa 30s)

6. Tráfego aumenta para 80% (pausa 30s)

7. Promoção final para 100%

#### Ajuste Manual de Tráfego (Demonstração)

Durante a apresentação, demonstrar o controle de tráfego:

```bash
# Iniciar rollout
kubectl apply -f rollout-prod.yaml

# Ver status atual
kubectl argo rollouts get rollout ivodocker-prod -n prod

# Definir 15% de tráfego manualmente
kubectl argo rollouts set canary ivodocker-prod -n prod --weight 15

# Aumentar para 50%
kubectl argo rollouts set canary ivodocker-prod -n prod --weight 50

# Aumentar para 75%
kubectl argo rollouts set canary ivodocker-prod -n prod --weight 75

# Promover completamente
kubectl argo rollouts promote ivodocker-prod -n prod --full
```

## Dashboard do Argo Rollouts

Para visualizar graficamente:

```bash
kubectl argo rollouts dashboard
```

Acesse: http://localhost:3100

## Verificação de Status

```bash
# Verificar todos os rollouts
kubectl get rollouts --all-namespaces

# Detalhes do rollout dev
kubectl describe rollout ivodocker-dev -n dev

# Detalhes do rollout prod
kubectl describe rollout ivodocker-prod -n prod

# Ver eventos
kubectl get events -n dev --sort-by='.lastTimestamp'
kubectl get events -n prod --sort-by='.lastTimestamp'
```

## Troubleshooting

### Rollout travado

```bash
# Abortar e fazer rollback
kubectl argo rollouts abort ivodocker-prod -n prod
kubectl argo rollouts undo ivodocker-prod -n prod
```

### Ver logs

```bash
# Logs do pod canary
kubectl logs -n prod -l app=ivodocker,env=prod -c ivodocker --tail=100

# Logs do pod stable
kubectl logs -n prod -l app=ivodocker,env=prod -c ivodocker --tail=100
```

### Reiniciar completamente

```bash
kubectl argo rollouts restart ivodocker-dev -n dev
kubectl argo rollouts restart ivodocker-prod -n prod
```

## Notas Importantes

1. **BlueGreen (dev)**: Sempre requer promoção manual (`autoPromotionEnabled: false`)
2. **Canary (prod)**: Primeira pausa é indefinida, demais são automáticas
3. Use `kubectl argo rollouts promote` para avançar nos steps
4. Use `kubectl argo rollouts abort` para cancelar e reverter
5. O dashboard visual ajuda muito na apresentação

## Referências

- [Argo Rollouts Documentation](https://argoproj.github.io/argo-rollouts/)
- [BlueGreen Strategy](https://argoproj.github.io/argo-rollouts/features/bluegreen/)
- [Canary Strategy](https://argoproj.github.io/argo-rollouts/features/canary/)
- [kubectl Plugin](https://argoproj.github.io/argo-rollouts/features/kubectl-plugin/)

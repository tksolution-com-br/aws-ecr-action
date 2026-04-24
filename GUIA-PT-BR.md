# Guia de Uso - Configuração por Ambiente

## Problema Resolvido

Esta implementação resolve o problema de **rate limiting do Docker Hub** ao fazer deploy, permitindo que você especifique apenas o ambiente (prd/hml/dev) e a action automaticamente usa a imagem correta do seu ECR privado.

## Como Funciona

### 1. Preparação das Imagens Base no ECR (Faça uma vez)

Primeiro, você precisa criar suas imagens Docker base no ECR:

```bash
# Login no ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# Pull da imagem original do Docker Hub (uma única vez)
docker pull docker:23.0.6

# Tag para suas imagens de PRD e HML
docker tag docker:23.0.6 123456789012.dkr.ecr.us-east-1.amazonaws.com/docker-base:prd
docker tag docker:23.0.6 123456789012.dkr.ecr.us-east-1.amazonaws.com/docker-base:hml

# Push para o ECR
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/docker-base:prd
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/docker-base:hml
```

### 2. Configuração no GitHub Actions (Simples!)

Agora você só precisa especificar o ambiente - a action cuida do resto!

#### Opção A: Workflows Separados por Ambiente (Recomendado)

**Workflow para Produção (.github/workflows/deploy-prd.yml):**

```yaml
name: Deploy to Production

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Build and Push to ECR
        uses: tksolution-com-br/aws-ecr-action@master
        with:
          access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          account_id: ${{ secrets.AWS_ACCOUNT_ID }}
          repo: minha-aplicacao
          region: us-east-1
          tags: latest,prd-${{ github.sha }}
          environment: prd
```

**Workflow para Homologação (.github/workflows/deploy-hml.yml):**

```yaml
name: Deploy to Homologation

on:
  push:
    branches:
      - develop
      - homolog

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Build and Push to ECR
        uses: tksolution-com-br/aws-ecr-action@master
        with:
          access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          account_id: ${{ secrets.AWS_ACCOUNT_ID }}
          repo: minha-aplicacao
          region: us-east-1
          tags: latest,hml-${{ github.sha }}
          environment: hml
```

#### Opção B: Workflow Único com Lógica Condicional

```yaml
name: Build and Deploy

on:
  push:
    branches:
      - main
      - develop

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Determine environment
        id: env
        run: |
          if [ "${{ github.ref }}" == "refs/heads/main" ]; then
            echo "name=prd" >> $GITHUB_OUTPUT
          else
            echo "name=hml" >> $GITHUB_OUTPUT
          fi

      - name: Build and Push to ECR
        uses: tksolution-com-br/aws-ecr-action@master
        with:
          access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          account_id: ${{ secrets.AWS_ACCOUNT_ID }}
          repo: minha-aplicacao
          region: us-east-1
          tags: latest,${{ steps.env.outputs.name }}-${{ github.sha }}
          environment: ${{ steps.env.outputs.name }}
```

### 3. Mapeamento Automático de Ambientes

A action mapeia automaticamente os ambientes para as imagens corretas:

| Ambiente                    | Imagem ECR Utilizada                                          |
| --------------------------- | ------------------------------------------------------------- |
| `prd`, `prod`, `production` | `{account_id}.dkr.ecr.{region}.amazonaws.com/docker-base:prd` |
| `hml`, `homolog`, `staging` | `{account_id}.dkr.ecr.{region}.amazonaws.com/docker-base:hml` |
| `dev`, `develop`            | `{account_id}.dkr.ecr.{region}.amazonaws.com/docker-base:hml` |
| Nenhum especificado         | `docker:23.0.6` (padrão do Docker Hub)                        |

### 4. Secrets Necessários no GitHub

Configure os seguintes secrets no seu repositório GitHub:

- `AWS_ACCESS_KEY_ID`: Sua chave de acesso AWS
- `AWS_SECRET_ACCESS_KEY`: Sua chave secreta AWS
- `AWS_ACCOUNT_ID`: ID da sua conta AWS (ex: 123456789012)
- `AWS_REGION`: Região AWS (ex: us-east-1)

## Vantagens

✅ **Elimina rate limiting**: Não depende mais do Docker Hub  
✅ **Configuração minimalista**: Apenas especifique `environment: prd` ou `environment: hml`
✅ **Centralizado**: URLs das imagens definidas internamente na action
✅ **Menos erros**: Não precisa digitar URLs completas manualmente
✅ **Controle total**: Você controla quando atualizar as imagens base  
✅ **Flexibilidade**: Imagens diferentes por ambiente  
✅ **Sem custos extras**: Usa o ECR que você já tem  
✅ **Mais rápido**: Imagens no ECR são geralmente mais rápidas de baixar  
✅ **Segurança**: Imagens privadas no seu ECR

## Uso Avançado

### Override da Imagem Base

Se precisar usar uma imagem diferente das configuradas internamente, você pode usar o parâmetro `base_image`:

```yaml
- uses: tksolution-com-br/aws-ecr-action@master
  with:
    # ... outros parâmetros ...
    base_image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/custom-docker:latest
```

Nota: Quando `base_image` é fornecido, o parâmetro `environment` é ignorado.

## Troubleshooting

### Erro: "unauthorized: authentication required"

Certifique-se de que as credenciais AWS têm permissão para acessar o ECR.

### Erro: "repository does not exist"

As imagens base precisam existir no ECR antes de usar. Execute os comandos da seção "Preparação das Imagens Base".

### Erro: "manifest unknown"

A tag especificada não existe. Verifique se você fez push da imagem com a tag correta (prd ou hml).

## Manutenção das Imagens Base

Recomenda-se atualizar suas imagens base periodicamente:

```bash
# Atualizar para nova versão do Docker
docker pull docker:24.0.0
docker tag docker:24.0.0 123456789012.dkr.ecr.us-east-1.amazonaws.com/docker-base:prd
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/docker-base:prd
```

## Rollback

Se houver problemas, você pode:

1. **Voltar ao Docker Hub**: Simplesmente remova o parâmetro `environment` do seu workflow
2. **Usar imagem customizada**: Use o parâmetro `base_image` para especificar uma imagem específica
3. **Manter a action sem mudanças**: A action é retrocompatível e continua funcionando sem o parâmetro `environment`

## Diferenças da Abordagem Anterior

**Antes** (passando URL completa):

```yaml
base_image: ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.us-east-1.amazonaws.com/docker-base:prd
```

**Agora** (apenas o ambiente):

```yaml
environment: prd
```

Muito mais simples e menos sujeito a erros! 🎉

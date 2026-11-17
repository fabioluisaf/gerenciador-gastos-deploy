# gerenciador-gastos-deploy

Manifests Kubernetes do gerenciador de gastos, aplicados pelo **Argo CD** (GitOps).
Este repositório não é editado na mão no dia a dia: quem altera as tags das imagens é o workflow **Release** dos repositórios de código.

| Ambiente | Namespace | Endereço | Perfil Spring |
|---|---|---|---|
| des | `gerenciador-gastos-des` | http://localhost:8081 | `dev` (Swagger em `/api/swagger`, SQL no log) |
| prod | `gerenciador-gastos-prod` | http://localhost:8082 | `prod` |

## Fluxo

```
Pipeline (manual, escolhe a branch)  ->  testes + imagem ghcr.io/...:sha-<commit>
Release  (manual, escolhe a branch e o ambiente)
   des:  aponta o overlay des para sha-<commit>
   prod: cria a tag X.Y.Z na imagem, aponta o overlay prod e cria a GitHub Release vX.Y.Z
Argo CD  ->  detecta o commit neste repositório e sincroniza o namespace
```

Rode a Pipeline e a Release separadamente em cada repositório (backend e frontend).

## Estrutura

```
base/              db (PostgreSQL), api (Spring Boot), web (nginx + Angular)
overlays/des/      namespace, tags das imagens, perfil dev, porta 8081
overlays/prod/     namespace, tags das imagens, perfil prod, porta 8082
argocd/            Applications do Argo CD (aplicadas uma única vez)
```

## Configuração inicial (uma vez)

### 1. GitHub

1. Crie o repositório público `gerenciador-gastos-deploy` e faça o push deste conteúdo na branch `main`.
2. Crie um **fine-grained personal access token** com acesso apenas ao repositório `gerenciador-gastos-deploy` e permissão **Contents: Read and write**.
3. Nos repositórios `gerenciador-gastos` e `gerenciador-gastos-frontend-angular`, crie o secret `DEPLOY_TOKEN` com esse token (*Settings > Secrets and variables > Actions*).
4. Depois da primeira Pipeline de cada repositório, deixe os pacotes públicos no GHCR (*perfil > Packages > pacote > Package settings > Change visibility*), para o cluster baixar as imagens sem credencial.
5. (Opcional) Em *Settings > Environments > prod* do backend e do frontend, adicione você como *required reviewer* para exigir aprovação antes de cada release em prod.

### 2. Cluster

Credenciais do banco (fora do git), uma por ambiente:

```bash
kubectl create namespace gerenciador-gastos-des
kubectl -n gerenciador-gastos-des create secret generic gerenciador-gastos-db \
  --from-literal=DB_USERNAME=postgres --from-literal=DB_PASSWORD='<senha-des>'

kubectl create namespace gerenciador-gastos-prod
kubectl -n gerenciador-gastos-prod create secret generic gerenciador-gastos-db \
  --from-literal=DB_USERNAME=postgres --from-literal=DB_PASSWORD='<senha-prod>'
```

> A senha só é gravada no Postgres na primeira inicialização do volume. Para trocá-la depois, altere-a no banco (`ALTER USER`) e no Secret.

Aplicações do Argo CD:

```bash
kubectl apply -f argocd/
```

## Argo CD

- Interface: https://localhost:9443 (certificado autoassinado)
- Usuário `admin`; senha inicial:

  ```bash
  kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
  ```

- Logs: abra a aplicação, clique no pod e vá na aba **Logs**.
- O Argo CD consulta este repositório a cada ~3 minutos; para aplicar na hora, clique em **Refresh**.
- Rollback: **History and rollback** na aplicação, ou reverta o commit de release neste repositório.

## Acesso ao banco

```bash
kubectl -n gerenciador-gastos-des port-forward svc/db 5434:5432
```

Conecte em `localhost:5434` com o usuário e a senha do Secret.

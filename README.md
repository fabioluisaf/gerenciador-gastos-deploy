# gerenciador-gastos-deploy

Manifests Kubernetes do ambiente de **produção** do gerenciador de gastos, aplicados pelo **Argo CD** (GitOps).
Este repositório não é editado na mão no dia a dia: quem altera as tags das imagens é o workflow **Release** dos repositórios de código.

- Namespace: `gerenciador-gastos-prod`
- Endereço: http://localhost:8082
- Perfil Spring: `prod`

O desenvolvimento roda na máquina local (run config da IDE + `npm start`), fora do Kubernetes.

## Fluxo

```
git tag X.Y.Z && git push origin X.Y.Z
Pipeline (manual, na tag X.Y.Z)  ->  testes + imagem ghcr.io/...:X.Y.Z
Release  (manual, na tag X.Y.Z)  ->  commit neste repositório + GitHub Release X.Y.Z
Argo CD                          ->  detecta o commit e atualiza o namespace
```

Rode a Pipeline e a Release separadamente em cada repositório (backend e frontend).
Rodar a Release em uma tag antiga faz rollback para aquela versão.

## Estrutura

```
base/              db (PostgreSQL), gastos-api (Spring Boot), web (nginx + Angular)
overlays/prod/     namespace, tags das imagens, perfil prod, porta 8082
argocd/            Application do Argo CD (aplicada uma única vez)
```

## Configuração inicial (uma vez)

### 1. GitHub

1. Crie um **fine-grained personal access token** com acesso apenas ao repositório `gerenciador-gastos-deploy` e a permissão **Contents** com acesso **Read and write**.
2. Nos repositórios `gerenciador-gastos` e `gerenciador-gastos-frontend-angular`, crie o secret `DEPLOY_TOKEN` com esse token (*Settings > Secrets and variables > Actions*).
3. Depois da primeira Pipeline de cada repositório, deixe os pacotes públicos no GHCR (*perfil > Packages > pacote > Package settings > Change visibility*), para o cluster baixar as imagens sem credencial.
4. (Opcional) Em *Settings > Environments > prod* do backend e do frontend, adicione você como *required reviewer* para exigir aprovação antes de cada release.

### 2. Cluster

Credenciais do banco (fora do git):

```bash
kubectl create namespace gerenciador-gastos-prod
kubectl -n gerenciador-gastos-prod create secret generic gerenciador-gastos-db \
  --from-literal=DB_USERNAME=postgres --from-literal=DB_PASSWORD='<senha>'
```

> A senha só é gravada no Postgres na primeira inicialização do volume. Para trocá-la depois, altere-a no banco (`ALTER USER`) e no Secret.

Aplicação do Argo CD:

```bash
kubectl apply -f argocd/
```

## Argo CD

- Interface: https://localhost:9443 (certificado autoassinado)
- Usuário `admin`; senha inicial:

  ```bash
  kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
  ```

- Logs: abra a aplicação, clique no pod e vá na aba **Logs**.
- O Argo CD consulta este repositório a cada ~3 minutos; para aplicar na hora, clique em **Refresh**.
- Rollback: **History and rollback** na aplicação, ou reverta o commit de release neste repositório.

## Acesso ao banco

```bash
kubectl -n gerenciador-gastos-prod port-forward svc/db 5434:5432
```

Conecte em `localhost:5434` com o usuário e a senha do Secret.

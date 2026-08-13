# gitops-config

## O que é este repositório

Repositório GitOps que define **o estado desejado** do cluster kind (`kind-kind`) e dos aplicativos nele, gerenciado por **Flux v2**. Nenhum manifest aqui é aplicado manualmente — o Flux reconcilia este repositório continuamente e aplica/prune as mudanças.

## Fluxos de trabalho

Flux é bootstrapped com **dois caminhos**:

1. `clusters/dev/flux-system/` — os próprios componentes do Flux (instalados pelo `flux bootstrap`, **não editar manualmente**)
   - `gotk-components.yaml` — CRDs + controllers do Flux
   - `gotk-sync.yaml` — `GitRepository` (aponta para este repo, branch `main`) + `Kustomization` `flux-system`
2. `clusters/dev/kustomization.yaml` — o cluster: referencia os apps em `apps/`

O `Kustomization` `flux-system` reconcilia o path `./clusters/dev` a cada `10m` (interval do `gotk-sync.yaml`). O `GitRepository` poll o repo a cada `1m`.

## Estrutura

```
gitops-config/
├── clusters/
│   └── dev/
│       ├── kustomization.yaml          # cluster: referencia ../../apps/*
│       └── flux-system/                # gerado pelo flux bootstrap (não editar)
│           ├── gotk-components.yaml
│           ├── gotk-sync.yaml
│           └── kustomization.yaml
└── apps/
    ├── my-java-app1/                    # app Java (Helm chart OCI no GHCR)
    │   ├── kustomization.yaml
    │   ├── oci-repository.yaml         # HelmRepository type=oci → oci://ghcr.io/bazoocaze/charts
    │   └── helm-release.yaml           # HelmRelease com values (image tag, ingress, imagePullSecret)
    └── ingress-nginx/                  # ingress controller (Helm chart do repo público)
        ├── kustomization.yaml
        ├── helm-repository.yaml        # HelmRepository → https://kubernetes.github.io/ingress-nginx
        └── helm-release.yaml           # HelmRelease ingress-nginx (hostPort, DaemonSet, nodeSelector)
```

## Padrões usados

- **Um diretório por aplicativo em `apps/<nome>/`** com `kustomization.yaml` agrupando os recursos.
- **Apps adicionados ao cluster** referenciando `../../apps/<nome>` em `clusters/dev/kustomization.yaml`.
- **Helm via Flux** (`HelmRelease`), não via `helm install` manual. Charts do GHCR são `HelmRepository type: oci`; charts públicos usam `HelmRepository` regular.
- **Namespace padrão**: os HelmReleases usam `namespace: default` (secret `ghcr-auth` vive lá).
- **Intervalos**: `interval: 5m` nos HelmReleases, `10m`/`1h` nas sources.
- **DRY nos values do HelmRelease**: só declarar overrides que divergem dos defaults do chart (ex.: `image.repository`, `image.tag`, `imagePullSecrets`, `ingress.enabled: true`). Valores idênticos ao default do chart (ex.: `replicaCount`, `service.type`, `service.port`, `pullPolicy`, `ingress.hosts`) ficam no chart e não precisam ser repetidos — evita duplicação e mantém o HelmRelease enxuto. Isso TAMBÉM vale para o ingress: o HelmRelease só seta `enabled: true`; `className`, `hosts` e `paths` vêm dos defaults do chart.

## Ingress-nginx (via Flux)

- **Namespace**: `default`. Chart oficial `ingress-nginx/ingress-nginx` (HelmRepository público).
- **kind**: `kind: DaemonSet` com `hostPort.enabled: true`, nodeSelector `kubernetes.io/hostname: kind-control-plane` e **toleration** `node-role.kubernetes.io/control-plane: NoSchedule` (o control-plane do kind é tainted).
- O `extraPortMappings` (80/443) vive em `kind/kind-config.yaml` — não é gerenciado pelo Flux.
- Acesso externo: `http://localhost/hello` (porta 80 mapeada pelo kind).

## ⚠️ Migrar recurso manual → GitOps: ordem obrigatória

Quando um recurso foi instalado manualmente (`kubectl apply`) e passa a ser gerenciado por Flux, **NUNCA remova o manual antes de o Flux aplicar o novo**. Ordem correta:

1. Aplicar o novo recurso via Flux primeiro (`flux reconcile source git flux-system` + `flux reconcile kustomization flux-system -n flux-system`)
2. Validar que o novo subiu (pod Ready, recursos cluster-escopados recriados)
3. Só então remover a instalação manual

**Motivo (incidente 03/08/2026):** o manifesto manual do ingress-nginx e o Helm chart criavam recursos cluster-escopados com os MESMOS nomes (ClusterRole `ingress-nginx`, IngressClass `nginx`, ValidatingWebhookConfiguration `ingress-nginx-admission`). Remover o manual apagou recursos que o Flux já geria → upgrade Helm travou com `NotFound`.

**Se um HelmRelease ficar com upgrade quebrado:**
1. `flux suspend helmrelease <name> -n <ns>` — para o Flux tentar
2. Remover órfãos do release (ex.: `kubectl delete secret ingress-nginx-admission`)
3. `helm uninstall <name> -n <ns>` — limpa o release
4. `flux resume helmrelease <name> -n <ns>` — Flux reinstala limpo (revision 1)

## Comandos úteis

```bash
# Flux
flux check                                    # saúde dos controllers
flux get kustomizations                       # status do Kustomization flux-system
flux get helmreleases -A                      # listar helm releases
flux get sources helm -A                      # listar helm sources

# Forçar reconciliação
flux reconcile source git flux-system         # puxa o commit mais recente do repo (1° passo sempre)
flux reconcile kustomization flux-system -n flux-system   # aplica o estado (2° passo)

# Útil: kubectl kustomize do cluster pra validar antes de commitar
kubectl kustomize clusters/dev
```

## Fluxo de rotação de credenciais GHCR (sem expor secrets)

Quando o usuário renova o token do GitHub, atualizar **3 pontos** na ordem:

1. **Docker local** (usado por `publish-image.sh`):
   ```bash
   gh auth token | docker login ghcr.io -u bazoocaze --password-stdin
   ```
2. **Helm local** (usado por `publish-chart.sh`):
   ```bash
   gh auth token | helm registry login ghcr.io -u bazoocaze --password-stdin
   ```
3. **`secret/ghcr-auth`** (namespace `default`) — recriar com dockerconfigjson **só** do ghcr.io (não copiar credenciais extras do `~/.docker/config.json`, ex.: ECR):
   ```bash
   python3 -c "import json; d=json.load(open('$HOME/.docker/config.json')); open('/tmp/ghcr-dockerconfig.json','w').write(json.dumps({'auths':{'ghcr.io':d['auths']['ghcr.io']}}))"
   chmod 600 /tmp/ghcr-dockerconfig.json
   kubectl create secret docker-registry ghcr-auth -n default --from-file=.dockerconfigjson=/tmp/ghcr-dockerconfig.json --dry-run=client -o yaml | kubectl apply -f -
   rm -f /tmp/ghcr-dockerconfig.json
   ```

**Regras da rotação:**
- Sempre usar `gh auth token | <cmd> --password-stdin` — o token nunca aparece em output
- Arquivos temporários com credenciais: `chmod 600` + remover após uso
- Após rotacionar, validar: `flux get sources helm -A` (Ready), `kubectl get helmcharts -A` (Ready), `curl http://localhost/hello` (HTTP 200)

## Fluxo de bump de versão de app

**Automático via Flux ImageUpdateAutomation:** o CI do `my-java-app1` publica imagem+chart no GHCR com versão `1.0.<run_number>`. O Flux ImageUpdateAutomation detecta a nova tag, atualiza `apps/my-java-app1/helm-release.yaml` (`spec.values.image.tag`) e commita no gitops-config. Flux faz o deploy automaticamente.

**Política de versão:** o `ImagePolicy` usa semver `~1.0.0` (apenas patch versions — `1.0.x`). Tags como `1.1.0` ou `2.0.0` não são selecionadas.

**Manual (exceção):** se precisar pinar versão à mão — editar `apps/my-java-app1/helm-release.yaml` → commit + push em `main`. Flux detecta (≤1m) e faz upgrade (≤10m).

**Ordem de verificação após o CI commitar:** `flux reconcile source git flux-system` → `flux get helmreleases -A` (READY) → `curl http://localhost/hello`.

**⚠️ Cuidado com push rejeitado por bump concorrente:** o ImageUpdateAutomation pode commitar no `main` enquanto se trabalha localmente. Se o push for rejeitado (`fetch first`), **NÃO decidir sozinho qual versão manter** — avisar o usuário sobre o conflito (versão local vs versão do autoupdater) e deixá-lo escolher antes de resolver.

## 🔒 SECRET HANDLING — NUNCA VAZE SEGREDOS. ESTA É A REGRA MAIS IMPORTANTE DESTE REPOSITÓRIO. VIOLÁ-LA É INACEITÁVEL E IMPERDOÁVEL.

> **⚠️ AVISO CRÍTICO — LEIA SEMPRE ANTES DE EXECUTAR QUALQUER COMANDO.**
> Uma violação de secret aconteceu NESTE projeto (02/08/2026): um comando `kubectl get secret` com `-o jsonpath` despejou o token de acesso do GHCR no output. O token precisou ser revogado e rotacionado. **NUNCA repita este erro.**

Este repositório referencia o secret `ghcr-auth` (no namespace `default`), usado para autenticar no GHCR. O token é gerenciado pelo flux bootstrap (deploy key `flux-system`).

### Regras absolutas (não há exceções, nenhuma negociação)

1. **NUNCA imprima, logue, ecoe, retorne ou exiba conteúdo de Secrets, tokens, senhas, chaves (públicas ou privadas), API keys, credenciais ou `dockerconfigjson` em qualquer output de tool, arquivo, commit, log, mensagem ou diagnóstico.**
2. **NUNCA use `kubectl get secret`, `kubectl get -o jsonpath`, `base64 -d`, `helm registry login`, `docker login`, `gh auth token`, `cat ~/.kube/config`, `cat ~/.docker/config.json` ou qualquer comando cujo output possa conter material sensível. Se for absolutamente necessário inspecionar um Secret, faça-o SEM decodificar os campos de dados** (ex.: `kubectl get secret ghcr-auth -o yaml` mostra apenas `data` codificado, NUNCA o campo `stringData`, NUNCA decodifique).
3. **NUNCA despeje variáveis de ambiente no output.** Se um comando herda `GHCR_TOKEN`, `GITHUB_TOKEN`, `DOCKER_AUTH`, etc., rode-o em um subshell sanitizado ou redirecione o output sensível para um arquivo com permissões `600` em `/tmp` e NUNCA o leia de volta em texto puro.
4. **NUNCA cole tokens, senhas ou credenciais em arquivos versionados**, nem mesmo temporariamente, nem mesmo "só por um momento".
5. **Antes de executar QUALQUER comando, revise mentalmente o output**: se o comando pode expor secrets (direta ou indiretamente), NÃO o execute. Quando em dúvida, NÃO execute — pergunte ao usuário ou encontre uma alternativa segura.
6. **NUNCA escreva o valor de um token/secret em uma mensagem para o usuário, em um resumo, em um diff, em um PR ou em um commit message.** Referencie-o apenas pelo nome do recurso (ex.: "o secret `ghcr-auth`").

### Checklist obrigatório antes de rodar comandos de inspeção no cluster

- O comando decodifica algo? → **NÃO RODE**.
- O comando tem `-o jsonpath`, `-o go-template`, `-o yaml` sobre Secret? → **NÃO RODE** (ou use apenas campos estruturais, nunca `data`/`stringData`).
- O output pode conter um token regex-like (`gho_`, `ghp_`, `ghs_`, `ghu_`, `AKIA`, `BEGIN ... PRIVATE KEY`, etc.)? → **NÃO RODE**.
- Se precisar verificar autenticação/credenciais, use comandos que NÃO retornam o segredo (ex.: `docker login` em modo interativo, `gh auth status`, `flux get sources`).

### Se, apesar de tudo, um secret for exposto

1. **AVISE O USUÁRIO IMEDIATAMENTE** e com clareza.
2. **NÃO continue o trabalho** até o usuário revogar/rotacionar a credencial.
3. **NUNCA tente "consertar" silenciosamente** reescrevendo o histórico ou o log — o segredo já foi comprometido e precisa ser rotacionado pelo usuário.
4. Depois de rotacionado, registre o incidente aqui (seção acima) para que o próximo agente aprenda.

## Agent Behavior

- Quando o usuário pede **discussão, avaliação ou review**, o agente deve primeiro discutir e só fazer mudanças após o usuário autorizar explicitamente.
- **Sempre** validar com `kubectl kustomize clusters/dev` antes de commitar mudanças estruturais.
- Após alterar o repo, seguir a ordem de reconciliação: `flux reconcile source git flux-system` primeiro, depois `flux reconcile kustomization flux-system -n flux-system`.
- **NUNCA** editar `clusters/dev/flux-system/` (gerado pelo bootstrap).
- Commits do usuário **`CI Release Bot`** (`chore: bump my-java-app1 to <version> [skip ci]`) são gerados automaticamente pelo CI do `my-java-app1` — não desfazer e não se surpreender com eles.
- Commits em `main` disparam reconciliação automática — mensagens de commit claras e sem secrets (ver regras acima).

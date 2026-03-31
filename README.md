# 🎓 GitHub Actions — Laboratório Didático

Estrutura completa e comentada para aprender **Reusable Workflows**,
**Composite Actions**, inputs/outputs, contexts, secrets e muito mais.

---

## 📁 Estrutura de Arquivos

```
.github/
├── workflows/
│   ├── ci.yml                     ← Entry point (caller)
│   ├── reusable-build-test.yml    ← Reusable: lint + test + build
│   ├── reusable-deploy.yml        ← Reusable: deploy + summary
│   └── extras-conceitos.yml       ← Playground de conceitos avançados
│
└── actions/
    ├── setup-node/
    │   └── action.yml             ← Composite: Node.js + cache
    └── deploy/
        └── action.yml             ← Composite: deploy + health check
```

---

## 🧠 Os Dois Grandes Conceitos

### Reusable Workflow

Um arquivo `.yml` normal, mas com `on: workflow_call`.
Chamado por outro workflow via `uses:`.

```yaml
# No caller (ci.yml):
jobs:
  build:
    uses: ./.github/workflows/reusable-build-test.yml
    with:
      node-version: "20"       # passa inputs
    secrets: inherit           # repassa secrets

# No reusable (reusable-build-test.yml):
on:
  workflow_call:
    inputs:
      node-version:
        type: string
        default: "20"
```

**Use quando:** precisa encapsular **múltiplos jobs** ou jobs em **runners diferentes**.

---

### Composite Action

Arquivo `action.yml` em `.github/actions/<nome>/`.
Encapsula **steps** reutilizáveis dentro de um job.

```yaml
# No workflow:
steps:
  - uses: ./.github/actions/setup-node
    with:
      node-version: "20"

# No action.yml:
runs:
  using: "composite"
  steps:
    - name: Instalar Node
      shell: bash          # obrigatório em composite!
      run: echo "setup..."
```

**Use quando:** tem steps repetitivos (setup, auth, cache) que aparecem em vários jobs.

---

## 🔄 Fluxo Completo do Pipeline

```
Push na main / workflow_dispatch
          │
          ▼
      ci.yml
          │
          ├──► reusable-build-test.yml
          │         ├─ Job: lint  ─────────────────────┐ paralelo
          │         ├─ Job: test (matrix: 18, 20)  ────┤
          │         └─ Job: build  ◄───────────────────┘
          │                  │
          │                  ├─ usa composite: setup-node
          │                  └─ output: artifact-name, build-version
          │
          └──► reusable-deploy.yml  (needs: build-and-test)
                    └─ Job: deploy
                             ├─ usa composite: deploy
                             └─ output: deploy-url
```

---

## 📌 Inputs & Outputs: Como Fluem

```
step  →  $GITHUB_OUTPUT  →  job.outputs  →  workflow.outputs  →  needs.<id>.outputs
```

**1. Step define um output:**
```bash
echo "minha-chave=meu-valor" >> $GITHUB_OUTPUT
```

**2. Job declara que expõe:**
```yaml
jobs:
  meu-job:
    outputs:
      minha-chave: ${{ steps.meu-step.outputs.minha-chave }}
```

**3. Reusable workflow expõe para o caller:**
```yaml
on:
  workflow_call:
    outputs:
      minha-chave:
        value: ${{ jobs.meu-job.outputs.minha-chave }}
```

**4. Caller lê:**
```yaml
${{ needs.build-and-test.outputs.minha-chave }}
```

---

## 🌐 Contexts Mais Usados

| Expressão | O que retorna |
|---|---|
| `github.sha` | Hash completo do commit |
| `github.ref_name` | Nome da branch ou tag |
| `github.actor` | Usuário que disparou |
| `github.event_name` | `push`, `pull_request`, `workflow_dispatch`… |
| `github.run_number` | Número sequencial do run |
| `github.run_id` | ID único do run |
| `runner.os` | `Linux`, `Windows`, `macOS` |
| `job.status` | `success`, `failure`, `cancelled` |

---

## ⚙️ Variáveis Especiais do Runner

| Variável | Para que serve |
|---|---|
| `$GITHUB_OUTPUT` | Setar output de um step |
| `$GITHUB_ENV` | Setar env var para steps seguintes |
| `$GITHUB_STEP_SUMMARY` | Escrever Markdown na aba Summary |
| `$GITHUB_PATH` | Adicionar diretório ao PATH |

---

## ⚡ Cheat Sheet Rápido

```yaml
# Condicionais
if: github.ref == 'refs/heads/main'
if: startsWith(github.ref, 'refs/heads/feature/')
if: always()       # roda mesmo se job anterior falhou
if: failure()      # só se houve falha
if: cancelled()    # só se foi cancelado

# Cache
- uses: actions/cache@v4
  with:
    path: node_modules
    key: npm-${{ hashFiles('package-lock.json') }}

# Matrix
strategy:
  matrix:
    node: [18, 20]
    os: [ubuntu-latest, windows-latest]
  fail-fast: false

# Timeout
timeout-minutes: 10          # no job
timeout-minutes: 2           # no step

# Não bloquear em falha
continue-on-error: true

# Concurrency (evitar runs paralelos)
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: true

# Artefatos
- uses: actions/upload-artifact@v4
  with:
    name: meu-build
    path: dist/
    retention-days: 7

- uses: actions/download-artifact@v4
  with:
    name: meu-build
    path: dist/
```

---

## 🗺️ Próximos Passos

1. **Environments com aprovação manual** → Settings → Environments → Required reviewers
2. **Secrets por ambiente** → cada environment tem seus próprios secrets
3. **Reusable de outro repositório** → `uses: org/repo/.github/workflows/arquivo.yml@main`
4. **Actions do Marketplace** → [github.com/marketplace](https://github.com/marketplace?type=actions)
5. **Self-hosted runners** → `runs-on: self-hosted`

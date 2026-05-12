# Gerenciador de Snapshots — PMI OceanPact × CBO

Ferramenta HTML para gerenciamento das snapshots versionadas do Hub Multi-FT PMI OceanPact × CBO. Permite listar, ativar, fazer upload e excluir snapshots diretamente no bucket S3, sem precisar acessar o console AWS.

---

## Funcionalidades

- **Listar snapshots** — exibe todas as versões disponíveis no bucket S3 com status (ativa/inativa), tamanho e data
- **Tornar ativa** — marca qualquer snapshot como a versão padrão servida pelo Hub, com modal de confirmação
- **Upload de nova snapshot** — envio direto para o S3 via presigned URL, sem limite de tamanho, com barra de progresso
- **Excluir snapshot** — remove versões antigas com proteção contra exclusão da snapshot ativa

---

## Arquitetura

```
Browser (Gerenciador HTML)
    │
    ├── GET  /snapshots              → lista versões + identifica ativa
    ├── POST /snapshots/apply        → ativa ou atualiza snapshot existente
    ├── POST /snapshots/upload       → obtém presigned URL + confirma upload
    └── DELETE /snapshots/apply      → exclui snapshot do S3

API Gateway HTTP API (hgelha26b4)
    ├── Lambda pmi-list-snapshots    → GET /snapshots
    ├── Lambda pmi-get-snapshot      → GET /snapshots/data
    ├── Lambda pmi-apply-snapshot    → POST + DELETE /snapshots/apply
    └── Lambda pmi-upload-url        → POST /snapshots/upload

S3: gradus-pmi-snapshots
    ├── snapshots/snapshot_v31.json
    ├── snapshots/snapshot_v30.json
    └── active.txt  ← key da snapshot ativa
```

---

## Fluxo de Upload

O upload usa **presigned URL** para contornar o limite de 10 MB do API Gateway:

1. Ferramenta envia `action=get-upload-url` ao Lambda `pmi-upload-url`
2. Lambda gera uma presigned URL válida por 5 minutos e retorna o `key` do arquivo no S3
3. Ferramenta faz `PUT` direto no S3 via presigned URL (sem passar pelo API Gateway)
4. Ferramenta envia `action=confirm-upload` ao Lambda para registrar e ativar se necessário

---

## Infraestrutura AWS

### Lambdas

**`pmi-apply-snapshot`** — ativa, atualiza e exclui snapshots

| action | descrição |
|--------|-----------|
| `update` | atualiza snapshot existente via merge; se `diff={}`, apenas ativa |
| `clone` | clona snapshot base aplicando diff |
| `DELETE` (HTTP) | exclui snapshot; impede exclusão da ativa |

**`pmi-upload-url`** — gerencia upload direto ao S3

| action | descrição |
|--------|-----------|
| `get-upload-url` | gera presigned URL para PUT no S3 |
| `confirm-upload` | verifica que o arquivo chegou e ativa se `set_active=true` |

### Variáveis de ambiente dos Lambdas

| Variável | Valor |
|----------|-------|
| `PMI_BUCKET` | `gradus-pmi-snapshots` |
| `PMI_API_TOKEN` | `pmi-token-2026` |

### Rotas do API Gateway

| Método | Rota | Lambda |
|--------|------|--------|
| GET | `/snapshots` | `pmi-list-snapshots` |
| GET | `/snapshots/data` | `pmi-get-snapshot` |
| POST | `/snapshots/apply` | `pmi-apply-snapshot` |
| DELETE | `/snapshots/apply` | `pmi-apply-snapshot` |
| OPTIONS | `/snapshots/apply` | `pmi-apply-snapshot` |
| POST | `/snapshots/upload` | `pmi-upload-url` |
| OPTIONS | `/snapshots/upload` | `pmi-upload-url` |

### CORS

**API Gateway:** configurado com `*` para Allow-Origin, métodos `GET POST DELETE OPTIONS`, headers `content-type, x-api-token`.

**S3 bucket:**
```json
[{
  "AllowedHeaders": ["*"],
  "AllowedMethods": ["GET", "PUT", "POST", "DELETE", "HEAD"],
  "AllowedOrigins": ["*"],
  "ExposeHeaders": []
}]
```

---

## Hospedagem no PPR

O arquivo `pmi-snapshot-manager.html` é servido como ferramenta freeform no PPR via nginx:

```nginx
location /hub-manager/ {
    alias "/opt/gradus/consulting-kb/2. Application/src/public/";
    index pmi-snapshot-manager.html;
    try_files $uri pmi-snapshot-manager.html =404;
}
```

URL no PPR: `https://pprdev.gradusanalytics.com.br/hub-manager/`

---

## Variáveis no HTML

```javascript
const API   = 'https://hgelha26b4.execute-api.us-east-1.amazonaws.com';
const TOKEN = 'pmi-token-2026';
```

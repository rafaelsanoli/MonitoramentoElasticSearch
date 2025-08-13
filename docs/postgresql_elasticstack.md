# Guia de Implantação — Logs do PostgreSQL no Elastic Stack

> Tutorial detalhado para configurar a integração completa de logs do PostgreSQL com Elastic Stack, incluindo dicas para resolução de erros comuns.

---

## 1. Visão Geral

**Fluxo:** PostgreSQL (logs) → Filebeat (módulo `postgresql`) → Ingest Pipeline (Elasticsearch) → Data Streams/ILM → Kibana.

**Opcional:** Filebeat → Logstash → Ingest Pipeline → Elasticsearch (para máscara/enriquecimento).

**Diretrizes:**

- Índices diários.
- 1 shard primário / 1 réplica.
- Retenção 30–90 dias.
- Campos extras para `env`, `db_cluster` e `tenant`.

---

## 2. Pré-requisitos

- **Docker/Docker Compose** instalados (ou cluster ES existente).
- **Acesso root** nos hosts PostgreSQL para instalar Filebeat.
- Portas abertas:
  - ES: `9200` (HTTP), `9300` (transport)
  - Kibana: `5601`
  - Logstash (se usar): `5044`
- PostgreSQL configurado para gerar logs com `log_line_prefix` compatível com o módulo.

---

## 3. Subir Elasticsearch e Kibana

1. Copiar `.env.example` para `.env` e ajustar variáveis.
2. **Ambiente Dev** (sem TLS):
   ```bash
   cd infra
   docker compose --profile dev up -d
   ```
3. **Ambiente Prod** (com TLS e RBAC):
   - Inserir certificados em `infra/certs/`.
   - Ativar TLS e segurança no `docker-compose.yml`.
   - Subir com:
     ```bash
     docker compose --profile prod up -d
     ```

---

## 4. Criar ILM, Template e Pipeline

**Política ILM**:

```bash
curl -X PUT $ES_HOST/_ilm/policy/logs-postgresql-policy \
  -H 'Content-Type: application/json' \
  -d @config/ilm/policy-logs-postgresql.json
```

**Index Template**:

```bash
curl -X POST $ES_HOST/_index_template \
  -H 'Content-Type: application/json' \
  -d @config/ilm/template-logs-postgresql.json
```

**Ingest Pipeline**:

```bash
curl -X PUT $ES_HOST/_ingest/pipeline/pipeline-postgresql \
  -H 'Content-Type: application/json' \
  -d @config/ingest/pipeline-postgresql.json
```

---

## 5. Configurar PostgreSQL para Logs

No `postgresql.conf`:

```conf
logging_collector = on
log_destination = 'stderr'
log_directory = 'pg_log'
log_filename = 'postgresql-%Y-%m-%d.log'
log_line_prefix = '%m [%p] %q%u@%d '
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0
log_autovacuum_min_duration = 0
log_min_error_statement = error
log_duration = on
```

Reinicie o PostgreSQL após a alteração.

---

## 6. Instalar e Configurar Filebeat

**Instalação**:

```bash
sudo apt-get install filebeat
```

**Ativar módulo PostgreSQL**:

```bash
sudo filebeat modules enable postgresql
```

**Configuração básica (**``**)**:

```yaml
filebeat.modules:
  - module: postgresql
    log:
      enabled: true
      var.paths: ["/var/log/postgresql/*.log"]
      input:
        fields:
          env: "prod"
          db_cluster: "clusterA"
          tenant: "clienteX"

processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~

setup.template.enabled: false
setup.template.type: data_stream
setup.ilm.enabled: true
setup.ilm.policy_name: "logs-postgresql-policy"
setup.ilm.rollover_alias: "logs-postgresql"
setup.ilm.pattern: "{now/d}-000001"

output.elasticsearch:
  hosts: ["http://SEU_ES:9200"]
  username: "elastic"
  password: "changeme"
  pipeline: "pipeline-postgresql"
```

**Habilitar serviço**:

```bash
sudo systemctl enable --now filebeat
```

---

## 7. (Opcional) Usar Logstash

- Subir com:

```bash
cd infra
docker compose --profile with-logstash up -d
```

- Alterar saída no Filebeat para:

```yaml
output.logstash:
  hosts: ["SEU_LOGSTASH:5044"]
```

---

## 8. Criar Data View no Kibana

1. Acessar `http://SEU_KIBANA:5601`.
2. Importar `config/examples/kibana-data-view.ndjson`.
3. Usar Discover para explorar logs.

Exemplos de filtros KQL:

- `log.level : "error"`
- `postgresql.log.database : "meubanco" and postgresql.log.user : "app_user"`
- `message : "*duration*" and log.level : ("notice" or "info")`

---

## 9. Validação

- Checar saúde do cluster:

```bash
curl -s $ES_HOST | jq
```

- Confirmar data streams:

```bash
curl -s $ES_HOST/_data_stream/logs-postgresql* | jq
```

- Verificar chegada de documentos:

```bash
curl -s "$ES_HOST/logs-postgresql-*/_search?q=event.dataset:postgresql.log&size=1" | jq
```

---

## 10. Dicas para Erros Comuns

**Nada chega no ES**

- Verificar conexão (`curl $ES_HOST` do host Filebeat).
- Conferir usuário/senha.

**Falha no parse**

- Checar `log_line_prefix` e caminho dos logs (`var.paths`).

**ILM/Template não aplicados**

- Reaplicar JSONs e confirmar com:

```bash
GET _ilm/policy/logs-postgresql-policy
GET _index_template
```

**Pipeline não encontrado**

- Garantir que `pipeline-postgresql` está definido no output.

---

## 11. Checklist Final

```markdown
# Checklist Final — Projeto de Integração PostgreSQL + Elastic Stack

- [ ] **Elasticsearch e Kibana** em execução e acessíveis.
- [ ] **ILM, Template e Pipeline** criados e aplicados sem erros.
- [ ] **PostgreSQL** configurado para gerar logs (`logging_collector`, `log_line_prefix`, etc.).
- [ ] **Filebeat** instalado e módulo PostgreSQL habilitado.
- [ ] **Campos de metadados** (`env`, `db_cluster`, `tenant`) definidos corretamente no Filebeat.
- [ ] **Logs recebidos** no Elasticsearch e visíveis via Discover no Kibana.
- [ ] **Data View** criada no Kibana para `logs-postgresql-*`.
- [ ] **Rollover diário** funcionando conforme política ILM.
- [ ] **Retenção** configurada (30–90 dias) e confirmada.
- [ ] **Segurança (TLS e RBAC)** implementada no ambiente de produção.
- [ ] **Documentação** atualizada e versionada no repositório.
- [ ] **Consulta de teste** executada no PostgreSQL e visualizada no Kibana.
- [ ] **Validação da política ILM** feita com `_ilm/explain`.
```
---



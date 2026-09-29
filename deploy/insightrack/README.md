# Evolution Go 0.7.2 + whatsmeow atualizado — runbook InsighTrack

Imagem: `ghcr.io/insightrack/evolution-go:0.7.2-wm-20260929`
(gerada por `.github/workflows/publish_ghcr_image.yml`; a data no tag é a data UTC do build —
confira o tag exato no resumo do run do Actions antes de usar os comandos abaixo).

Arquivos:

| Arquivo | Uso |
|---|---|
| `stack-evolution-go-test.yml` | Stack de teste (`evolutiongo-test.tracklabs.com.br`, bancos `*_test`) |
| `stack-evolution-go-prod.yml` | Stack de produção com a imagem nova e sem `max_attempts` |

Antes de usar, troque os placeholders `SENHA_AQUI` e `TROCAR_GLOBAL_API_KEY_*` pelos valores reais
(no servidor, nunca no repositório).

Variáveis usadas nos comandos (ajuste ao seu ambiente; confira com `docker stack ls` e `docker service ls`):

```bash
PROD_STACK=evolution_go                        # nome do stack de produção
PROD_SVC=${PROD_STACK}_evolution_go            # <stack>_<service>
TEST_STACK=evolution_go_test
TEST_SVC=${TEST_STACK}_evolution_go_test
NEW_IMAGE=ghcr.io/insightrack/evolution-go:0.7.2-wm-20260929
OLD_IMAGE=evoapicloud/evolution-go:0.7.2
```

## 0. Acesso ao GHCR (uma vez, no manager)

O pacote no GHCR nasce privado. Ou torne-o público (GitHub → InsighTrack → Packages → evolution-go →
Package settings → Change visibility), ou faça login no manager com um PAT que tenha `read:packages`:

```bash
echo "$GHCR_PAT" | docker login ghcr.io -u <usuario-github> --password-stdin
docker pull "$NEW_IMAGE"
```

Com pacote privado, **sempre** use `--with-registry-auth` no `docker stack deploy` / `docker service update`.

## 1. Backup dos bancos (antes de qualquer mudança)

Rodar no nó onde está o container do Postgres. Sem `-t` no `docker exec` (TTY corrompe o dump binário).

```bash
PG=$(docker ps -q -f name=postgres | head -n1)   # confira que é o container certo: docker ps -f name=postgres
TS=$(date +%Y%m%d_%H%M%S)
mkdir -p ~/backups/evogo

docker exec "$PG" pg_dump -U postgres -d evogo_auth  -Fc > ~/backups/evogo/evogo_auth_${TS}.dump
docker exec "$PG" pg_dump -U postgres -d evogo_users -Fc > ~/backups/evogo/evogo_users_${TS}.dump

# conferir: tamanho > 0 e índice legível
ls -lh ~/backups/evogo/*_${TS}.dump
docker exec -i "$PG" pg_restore -l < ~/backups/evogo/evogo_auth_${TS}.dump  | head -20
docker exec -i "$PG" pg_restore -l < ~/backups/evogo/evogo_users_${TS}.dump | head -20
```

Restauração (só em desastre; pare o serviço antes):

```bash
docker service scale ${PROD_SVC}=0
docker exec -i "$PG" pg_restore -U postgres -d evogo_auth  --clean --if-exists < ~/backups/evogo/evogo_auth_<TS>.dump
docker exec -i "$PG" pg_restore -U postgres -d evogo_users --clean --if-exists < ~/backups/evogo/evogo_users_<TS>.dump
docker service scale ${PROD_SVC}=1
```

## 2. Stack de teste

Criar os bancos (o app tenta criar sozinho, mas só consegue se `evogo_rw` tiver `CREATEDB`):

```bash
docker exec "$PG" psql -U postgres -c 'CREATE DATABASE evogo_auth_test  OWNER evogo_rw;'
docker exec "$PG" psql -U postgres -c 'CREATE DATABASE evogo_users_test OWNER evogo_rw;'
```

Deploy e verificação:

```bash
docker stack deploy -c stack-evolution-go-test.yml --with-registry-auth ${TEST_STACK}
docker service ps ${TEST_SVC} --no-trunc
docker service logs -f ${TEST_SVC}
curl -s https://evolutiongo-test.tracklabs.com.br/server/ok          # {"status":"ok"}
curl -s https://evolutiongo-test.tracklabs.com.br/license/status
```

Observações:

- Bancos novos ⇒ a licença precisa ser ativada no manager do teste, como numa instalação nova.
- Use números de teste. Parear um número da produção no teste cria mais um aparelho conectado nele.
- O `REDIS_URL` é compartilhado, mas o código 0.7.2 não usa Redis (a variável não é lida), então não há colisão.
- Remover o teste depois: `docker stack rm ${TEST_STACK}` (volumes e bancos `*_test` ficam; apague à mão se quiser).

## 3. Update da produção

Com o backup do passo 1 feito e o teste validado:

```bash
docker stack deploy -c stack-evolution-go-prod.yml --with-registry-auth ${PROD_STACK}
docker service ps ${PROD_SVC} --no-trunc
docker service logs -f ${PROD_SVC}
curl -s https://evolutiongo.tracklabs.com.br/server/ok
curl -s -H "apikey: <GLOBAL_API_KEY>" https://evolutiongo.tracklabs.com.br/instance/all
```

O `stack deploy` aplica as duas mudanças: imagem nova e `restart_policy` sem `max_attempts`.
Com `replicas: 1` e o `update_config` padrão (`stop-first`), há alguns segundos de indisponibilidade e
as instâncias reconectam sozinhas (`CONNECT_ON_STARTUP=true`). Não use `start-first`: dois processos
com as mesmas sessões do WhatsApp ao mesmo tempo derrubam as conexões.

Na primeira subida, o whatsmeow novo migra o schema do `evogo_auth` da v14 para a v16, só adicionando
duas colunas com default (`whatsmeow_device.companion_meta_nonce`, `whatsmeow_chat_settings.wasa_root_secret_id`).

## 4. Rollback da produção

Volta só a imagem e mantém o `restart_policy` sem `max_attempts`:

```bash
docker service update --with-registry-auth --image ${OLD_IMAGE} ${PROD_SVC}
```

Depois, troque `image:` no `stack-evolution-go-prod.yml` para `evoapicloud/evolution-go:0.7.2`, para
que o próximo `stack deploy` não volte a subir a imagem nova.

Não precisa restaurar o banco: as migrações v15/v16 declaram compatibilidade a partir da v8, e o
0.7.2 original (schema v14) aceita o banco migrado. Restaure o dump só se houver corrupção de dados.

Alternativa: `docker service update --rollback ${PROD_SVC}` volta para a especificação anterior
inteira, o que também devolve o `max_attempts: 3`.

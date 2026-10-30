# Nha Karru: deploy

Configuração para correr o Nha Karru (frontend, backend e base de dados) com Docker, a partir das imagens publicadas no GitHub Container Registry. O servidor não precisa de código-fonte nem de compilar nada.

## Como funciona

```
Browser ──> frontend (nginx, porta HTTP_PORT)
                │  /api/*  (proxy interno)
                ▼
             backend (Adonis, porta 3333, não exposta)
                ▼
             db (PostgreSQL, volume pgdata, não exposta)
```

- Só o frontend publica uma porta. O backend e a base de dados ficam na rede interna do Docker.
- As imagens são construídas pelo GitHub Actions a cada push na `main` de cada repositório:
  - `ghcr.io/barle265-dev/nha-karru-backend`
  - `ghcr.io/barle265-dev/nha-karru-frontend`
- Todas as variáveis ficam num único ficheiro `.env`, que nunca vai para o git.

## Requisitos no servidor

- Docker com o plugin Compose (`docker compose version` deve funcionar)
- Git
- Uma porta livre para o frontend (por omissão 8080)
- Um token do GitHub para descarregar as imagens privadas (passo 2)

## Instalação

### 1. Obter esta configuração

```bash
git clone git@github.com:Barle265-dev/nha-karru-deploy.git
cd nha-karru-deploy
```

O repositório é privado. No servidor, o mais simples é gerar uma chave SSH (`ssh-keygen -t ed25519`) e adicionar a chave pública em **Settings → Deploy keys** deste repositório (só leitura). Em alternativa, copiar `docker-compose.yml` e `.env.example` com `scp`.

### 2. Login no registry de imagens

Criar um token em **GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)**, com o scope **`read:packages`** e nenhum outro, na conta `Barle265-dev`. Definir uma validade.

```bash
docker login ghcr.io -u Barle265-dev
```

Colar o token quando pedir `Password:`. Não o escrever no comando nem o guardar em ficheiros do repositório. Deve responder `Login Succeeded`.

### 3. Criar o `.env`

```bash
cp .env.example .env
```

Preencher no `.env`:

| Variável | O que pôr |
|---|---|
| `DB_PASSWORD` | Uma password forte, só para este servidor |
| `APP_KEY` | Uma chave nova, gerada com o comando abaixo |
| `APP_URL`, `CORS_ORIGIN`, `FRONTEND_URL` | O endereço público da app, por exemplo `http://IP_DO_SERVIDOR:8080` ou `https://nhakarru.exemplo.cv`. Os três com o mesmo valor |
| `HTTP_PORT` | A porta exposta no servidor. Se mudar, mude também o URL acima |
| `GOOGLE_CLIENT_ID` | O Client ID do Google, igual ao usado na build do frontend |
| `VAPID_*` | Chaves das notificações push. Vazio desliga o canal |
| `EMAIL_*` | Credenciais de email. Vazio desliga o canal |

Gerar o `APP_KEY` (não precisa de Node):

```bash
openssl rand -base64 48 | tr -dc 'A-Za-z0-9' | head -c 32; echo
```

Cada ambiente deve ter o seu `APP_KEY`. Não reutilizar o de desenvolvimento.

Se usar login Google, o endereço público (`APP_URL`) tem de estar nas **origens JavaScript autorizadas** do Client ID, na Google Cloud Console.

As variáveis `DB_HOST=db`, `DB_PORT`, `DB_USER`, `DB_DATABASE`, `HOST=0.0.0.0` e `PORT=3333` já vêm certas no exemplo e normalmente não se mexem.

### 4. Descarregar as imagens

```bash
docker compose pull
```

### 5. Criar a base de dados

A ordem importa. Num servidor novo o volume está vazio:

```bash
docker compose up -d db
docker compose run --rm migrate
docker compose run --rm migrate node ace db:seed
```

- `migrate` cria as tabelas.
- `db:seed` carrega as seguradoras (dados de referência de que a app precisa). É seguro repetir, não duplica. Em produção não cria utilizadores de teste.

### 6. Arrancar

```bash
docker compose up -d
docker compose ps
```

Devem aparecer `db` (healthy), `backend` e `frontend` como `Up`. O serviço `migrate` não aparece, porque só corre quando é pedido.

### 7. Verificar

```bash
curl -i http://localhost:8080/api/v1/
```

Responder com um JSON de erro vindo do Adonis (por exemplo 404) significa que o proxy chega ao backend. Depois, abrir o endereço da app no browser e criar uma conta.

## Atualizar para uma versão nova

Depois de um push na `main` do backend ou do frontend e de o workflow Docker ficar verde:

```bash
docker compose pull
docker compose run --rm migrate
docker compose up -d
```

O `migrate` só aplica as migrations novas. Se não houver, não faz nada.

## Voltar a uma versão anterior

Cada imagem é publicada com a tag `latest` e com uma tag `sha-xxxxxxx` (o commit). Para fixar uma versão, no `docker-compose.yml` trocar `:latest` por `:sha-xxxxxxx` nos serviços pretendidos e correr:

```bash
docker compose pull
docker compose up -d
```

As tags disponíveis veem-se em `github.com/Barle265-dev?tab=packages`. Se a versão antiga tiver menos migrations do que a base de dados, restaurar também um backup.

## Comandos do dia a dia

```bash
docker compose ps
docker compose logs -f backend
docker compose logs -f frontend
docker compose restart backend
docker compose stop
docker compose up -d
```

`docker compose down` remove os containers mas mantém os dados. **Nunca usar `down -v`**: apaga o volume `pgdata`, ou seja, toda a base de dados.

## Backup e restauro da base de dados

Backup:

```bash
docker compose exec -T db sh -c 'pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB"' > backup_$(date +%F).sql
```

Restauro numa base vazia (depois do passo 5, sem utilizadores criados):

```bash
docker compose exec -T db sh -c 'psql -U "$POSTGRES_USER" "$POSTGRES_DB"' < backup_AAAA-MM-DD.sql
```

Guardar os backups fora do servidor. Um backup que está só no mesmo disco não protege de nada.

## Problemas conhecidos

| Sintoma | Causa e solução |
|---|---|
| `unauthorized` ou `denied` no `pull` | Login em falta ou token errado. Repetir o passo 2. O token tem de ser da conta `Barle265-dev` e ter `read:packages` |
| `port is already allocated` | A porta já está ocupada por outro serviço. Mudar `HTTP_PORT` e o URL no `.env`. Ver quem a usa com `ss -ltnp` |
| `frontend` em `Restarting` e nos logs `host not found in upstream "backend"` | O container ficou sem rede. Recriar com `docker compose up -d --force-recreate frontend` |
| `502 Bad Gateway` | O backend não está a correr. Ver `docker compose logs backend` |
| O backend não arranca e fala numa variável | O `.env` está incompleto. Comparar com `.env.example` |
| Login Google não aparece ou falha | O `GOOGLE_CLIENT_ID` estava vazio quando a imagem do frontend foi construída (variável do repositório `VITE_GOOGLE_CLIENT_ID` no GitHub), ou a origem não está autorizada na Google Cloud Console |

## Antes de expor à internet

Ainda por tratar. Sem isto, não colocar a app acessível publicamente:

- **HTTPS.** O login Google e as notificações push exigem `https` fora de `localhost`. Colocar um proxy com certificado à frente do frontend (Caddy, Traefik ou o nginx do servidor).
- **IP real dos utilizadores.** Atrás do nginx do container, o backend vê sempre o mesmo IP, e o limitador de pedidos trata todos como uma só pessoa. Configurar o `trustProxy` no backend.
- **Backups automáticos** do volume `pgdata`.
- **Firewall.** Expor apenas as portas 80 e 443 (ou a `HTTP_PORT`) e a de SSH.

## Segurança

- O `.env` contém segredos e está no `.gitignore`. Nunca o commitar nem o enviar por chat ou email.
- Se um token ou chave aparecer em texto aberto, revogar e criar outro.
- Nenhum valor secreto deve estar neste repositório. Só o `.env.example`, com valores vazios ou de exemplo.

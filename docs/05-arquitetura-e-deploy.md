# Arquitetura e Deploy

Laboratório progressivo de infraestrutura com containers, proxy reverso, CI/CD e observabilidade. A VM é Ubuntu 24 provisionada com Vagrant.

## Objetivos

| Fase | Requisito | Prova de que está pronto | Status |
|------|-----------|--------------------------|--------|
| 1 | Dois servidores web em containers na VM | `docker compose ps` com app1 e app2 `healthy` | ✅ Feito |
| 2 | Proxy reverso L7 separado, roteamento por Host Header | `curl -H 'Host: app1.lab' localhost` devolve a app1 e `app2.lab` devolve a app2 | ✅ Feito |
| 3 | Pipeline GitHub Actions com self-hosted runner na VM | Um `git push` dispara o deploy na VM | 🔲 Pendente |
| 4 | Rollback integrado à pipeline | Uma versão quebrada enviada de propósito volta sozinha para a anterior | 🔲 Pendente |
| 5 | Monitoramento básico com alertas | Derrubar um container gera um alerta recebido | 🔲 Pendente |

---

## Fase 1 — Containers dos servidores web

### Conceitos

| Conceito | Analogia | No projeto |
|----------|----------|------------|
| Imagem | Cartucho lacrado, com etiqueta de versão | `nginx:1.27-alpine` |
| Container | O jogo rodando; descartável, recriado a partir da imagem | `app1`, `app2` |
| Volume | Cartão de memória plugado por fora | `./app1/html` montado dentro do container |

**Portas publicadas vs rede interna.** `ports: "8081:80"` liga uma porta da VM ao container. Sem `ports:`, o container só é alcançável por outros containers na mesma rede Docker. As apps não publicam portas de propósito: só o proxy fica exposto. A seta `->` na coluna PORTS do `docker ps` indica publicação.

**`running` vs `healthy`.** `running` significa que o processo está vivo. `healthy` significa que o healthcheck passou, ou seja, a aplicação responde. Um container pode estar `running` e `unhealthy`. O pipeline da Fase 4 deve esperar `healthy` (com timeout) e não declarar sucesso em `starting`.

**Tags fixas.** Nunca usar `latest`: ela muda sem aviso e impede voltar a uma versão conhecida.

**Projetos Compose separados.** Proxy e apps têm ciclos de vida independentes e se encontram pela rede externa `proxy-net`.

**Dados e containers.** O que está dentro do container sem volume some com `docker rm`. O que está em volume ou pasta montada sobrevive. A receita que recria tudo igual é o `docker-compose.yml`.

### Arquitetura atual

```
curl -H 'Host: app1.lab' ─┐
                          ├─► Traefik (porta 80 da VM) ─┬─ Host app1.lab → app1:80 (nginx)
curl -H 'Host: app2.lab' ─┘                             └─ Host app2.lab → app2:80 (Apache)
                          rede Docker compartilhada: proxy-net
```

Estrutura de diretórios na VM:

```
~/devops-lab/
├── proxy/
│   └── docker-compose.yml  # Traefik (projeto Compose separado)
└── apps/
    ├── docker-compose.yml  # app1 e app2
    ├── app1/html/index.html
    └── app2/html/index.html
```

Ambiente: VM Ubuntu 24 (Vagrant), Docker 29.9.3, Compose v5.7.1. Imagens: `traefik:v3.7`, `nginx:1.27-alpine`, `httpd:2.4-alpine`.

### Preparação da VM

```bash
# Docker Engine e plugin Compose
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER && newgrp docker

# Rede compartilhada entre proxy e apps
docker network create proxy-net

# Estrutura de diretórios
mkdir -p ~/devops-lab/{proxy,apps}
```

> Estar no grupo `docker` equivale a ser root na máquina, porque dá para montar `/` num container. Isso volta na Fase 3, com o runner.

### apps/docker-compose.yml

As linhas `labels:` foram acrescentadas na Fase 2.

```yaml
services:
  app1:
    image: nginx:1.27-alpine
    container_name: app1
    restart: unless-stopped
    volumes:
      - ./app1/html:/usr/share/nginx/html:ro
    healthcheck:
      test: ["CMD", "wget", "-qO", "/dev/null", "http://127.0.0.1/"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 5s
    networks: [proxy-net]
    labels:
      - traefik.enable=true
      - traefik.http.routers.app1.rule=Host(`app1.lab`)
      - traefik.http.routers.app1.entrypoints=web
      - traefik.http.services.app1.loadbalancer.server.port=80

  app2:
    image: httpd:2.4-alpine
    container_name: app2
    restart: unless-stopped
    volumes:
      - ./app2/html:/usr/local/apache2/htdocs:ro
    healthcheck:
      test: ["CMD", "wget", "-qO", "/dev/null", "http://127.0.0.1/"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 5s
    networks: [proxy-net]
    labels:
      - traefik.enable=true
      - traefik.http.routers.app2.rule=Host(`app2.lab`)
      - traefik.http.routers.app2.entrypoints=web
      - traefik.http.services.app2.loadbalancer.server.port=80

networks:
  proxy-net:
    external: true
```

| Trecho | Função |
|--------|--------|
| `image` com tag fixa | A versão do programa é conhecida e reproduzível |
| `restart: unless-stopped` | O container volta se cair ou se a VM reiniciar |
| `:ro` no volume | O container não altera o próprio conteúdo |
| `healthcheck` | O Docker passa a saber se a app responde. Usa `127.0.0.1` em vez de `localhost` para evitar resolução para IPv6 |
| Sem `ports:` | Nada é publicado diretamente na VM |
| `external: true` | A rede já existe; o Compose só se conecta a ela |

> O Apache usa a imagem `httpd` (e não `apache2`, que é o nome do pacote Debian) e outro docroot: `/usr/local/apache2/htdocs`.

### Liberar a porta 80 para o proxy

O nginx instalado na própria VM (pacote Ubuntu, versão 1.24.0) ocupava a porta 80. Foi verificado que o ngrok não rodava na VM e o serviço foi desabilitado:

```bash
pgrep -a ngrok                          # sem saída: ngrok não está rodando
sudo systemctl disable --now nginx
sudo ss -tlnp | grep ':80 '             # nada deve aparecer
```

O `--now` para o serviço imediatamente; o `disable` impede que volte no próximo boot.

### Validação da Fase 1

```bash
cd ~/devops-lab/apps
docker compose up -d
docker compose ps                       # app1 e app2 healthy, PORTS só com 80/tcp (sem seta)
docker run --rm --network proxy-net alpine wget -T 5 -qO- http://app1
docker run --rm --network proxy-net alpine wget -T 5 -qO- http://app2
```

O container descartável dentro da `proxy-net` acessa as apps pelo nome — o mesmo caminho que o proxy usa.

### Erros encontrados e o que ensinaram

| Sintoma | Causa | Aprendizado |
|---------|-------|-------------|
| `no configuration file provided: not found` | Comando rodado fora da pasta do compose | O Compose lê o arquivo da pasta atual; ler o prompt mostra onde você está |
| `wget: bad address 'app2'` | O container não existia | A rede só resolve nomes de containers que estão rodando nela |
| `curl: (6) Could not resolve host` | O nome não virou IP (digitação, DNS) | Erro (6) é de nome |
| `curl: (7) Failed to connect` | O nome resolveu, mas ninguém escuta na porta | Erro (7) é conexão recusada: nada escutando |
| `curl localhost:80` respondia `nginx/1.24.0 (Ubuntu)` | Quem respondia era o nginx da VM, não o app1 | Ler o header `Server:` diz quem respondeu |
| IP interno do container mudou entre execuções | O Docker entrega IPs por ordem de chegada | Nunca configurar por IP; usar o nome do container |

### Descoberta importante: o volume não tem versão

Editar o `index.html` da pasta montada muda o site sem recriar o container e sem mudar a tag da imagem. Trocar a tag de volta **não** reverte o conteúdo, porque ele está no volume e não na imagem. Consequência para a Fase 4: o conteúdo precisa estar dentro da imagem (Dockerfile com `COPY`), com tag por versão, para que trocar a tag troque o site junto.

---

## Fase 2 — Proxy reverso e roteamento por Host Header

### Conceito

Um proxy de Camada 7 decide lendo o conteúdo HTTP (o cabeçalho `Host`), e não só IP e porta. O `Host` é apenas um texto: a app o ignora, e quem lhe dá significado é o proxy. Sem o `-H`, o curl envia `Host: localhost`, que nenhuma regra reconhece, e o Traefik responde 404.

O Traefik tem duas necessidades com mecanismos diferentes:

| Precisa de | Mecanismo | O que faz |
|------------|-----------|-----------|
| Descobrir quem existe | `docker.sock` | Lê os containers e as labels |
| Entregar a requisição | `proxy-net` | Dá caminho de rede até o container e resolve o nome |

Sem a `proxy-net`, ele enxergaria as apps pelo socket, mas não teria caminho até elas.

### proxy/docker-compose.yml

```yaml
services:
  traefik:
    image: traefik:v3.7
    container_name: traefik
    restart: unless-stopped
    command:
      - "--providers.docker.exposedbydefault=false"
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
    ports:
      - "80:80"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks: [proxy-net]

networks:
  proxy-net:
    external: true
```

### Como as labels das apps funcionam

```
requisição → entrypoint (porta de entrada) → router (quando?) → service (para onde?)
```

| Label | Pergunta que responde |
|-------|-----------------------|
| `traefik.enable=true` | Quero ser roteada? (necessário porque o padrão é desligado) |
| `routers.app1.rule=Host(...)` | Quando a requisição é minha? Exige crase em volta do nome |
| `routers.app1.entrypoints=web` | Por qual porta de entrada aceito? Mesmo nome do `--entrypoints.web...` do proxy |
| `services.app1.loadbalancer.server.port=80` | Para qual porta interna do container encaminhar? |

O `app1` dentro de `routers.app1` e `services.app1` é um apelido e precisa ser único por app.

### Subir e validar

```bash
cd ~/devops-lab/proxy && docker compose config && docker compose up -d
cd ~/devops-lab/apps  && docker compose config && docker compose up -d

curl -s -H 'Host: app1.lab' localhost       # <h1>APP 1 (nginx)</h1>
curl -s -H 'Host: app2.lab' localhost       # <h1>APP 2 (apache)</h1>
curl -s -H 'Host: exemplo.lab' localhost    # 404 page not found
```

Resultado: os três testes retornaram o esperado. O mesmo endereço e a mesma porta responderam com duas apps diferentes, só pelo cabeçalho `Host`. O 404 sem HTML é do Traefik; um 404 do nginx viria como página HTML.

### Decisões de segurança

| Decisão | Por quê |
|---------|---------|
| `exposedbydefault=false` | Lista de permissão: só entra o container com `traefik.enable=true`. Com `true`, qualquer container novo na rede viraria rota pública sem ninguém pedir |
| Sem `api.insecure=true` e sem a porta 8080 | O dashboard sem autenticação lista rotas e nomes de containers, o que é reconhecimento gratuito para um atacante |
| Só a porta 80 publicada | Menor superfície de ataque |
| `:ro` no `docker.sock` é higiene, não proteção | `:ro` protege o arquivo do socket, mas o socket é a API do Docker: quem compromete o Traefik continua conseguindo criar containers, o que equivale a root na VM |
| Cuidado com firewall | O Docker publica portas via iptables e pode contornar o `ufw` |

Superfície da VM medida com `sudo ss -tlnp`:

- Antes do proxy: só `sshd` na porta 22.
- Depois: portas 22 e 80 (IPv4 e IPv6, via `docker-proxy`). A 8080 recusa conexão.

---

## Fase 3 — Pipeline CI/CD com GitHub Actions *(pendente)*

Uma pipeline de CI/CD é uma sequência automatizada de etapas que transforma um commit em código rodando em produção. No lugar de executar comandos manualmente na VM, cada `git push` dispara o GitHub Actions, que conecta na VM via self-hosted runner e executa o deploy. O benefício principal é consistência: o processo é sempre o mesmo, não depende de quem está na teclado nem do estado do terminal.

Self-hosted runner instalado na VM. Um `git push` dispara o deploy automaticamente sem intervenção manual.

---

## Fase 4 — Rollback integrado à pipeline *(pendente)*

Uma versão quebrada enviada de propósito deve ser detectada pela pipeline e revertida automaticamente para a versão anterior. Pré-requisito: conteúdo dentro da imagem (Dockerfile + `COPY`), não em volume, para que trocar a tag troque o site junto.

---

## Fase 5 — Monitoramento e alertas *(pendente)*

Monitoramento básico de infraestrutura. Derrubar um container deve gerar um alerta recebido.

---

## Cheat sheet de comandos

### Docker Compose *(rode dentro da pasta do projeto)*

| Comando | Para quê |
|---------|----------|
| `docker compose config` | Valida o arquivo e mostra como foi interpretado |
| `docker compose up -d` | Cria ou atualiza o que mudou (em segundo plano) |
| `docker compose ps` | Estado e health dos containers do projeto |
| `docker compose logs --tail 20 app2` | Últimas linhas de log de um serviço |
| `docker compose down` | Remove os containers (não apaga a pasta `html/` nem a rede externa) |

### Docker

| Comando | Para quê |
|---------|----------|
| `docker ps` / `docker ps -a` | Containers rodando / todos, inclusive parados |
| `docker port NOME` | Portas publicadas de um container (vazio = nenhuma) |
| `docker exec NOME comando` | Roda um comando dentro de um container |
| `docker rm NOME` | Remove um container (precisa estar parado) |
| `docker images` / `docker rmi IMAGEM` | Lista / remove imagens |
| `docker pull httpd:2.4-alpine` | Baixa uma imagem sem criar container |

### Rede e portas

| Comando | Para quê |
|---------|----------|
| `docker network ls` | Lista as redes Docker |
| `docker network inspect proxy-net` | Quem está na rede |
| `sudo ss -tulpn` | Portas TCP/UDP escutando na VM e o processo dono |

### Testes HTTP

| Comando | Para quê |
|---------|----------|
| `curl -I URL` | Só os headers (código, `Server:`) |
| `curl -s -H 'Host: app1.lab' localhost` | Testa a regra de Host no proxy |

### Sistema

| Comando | Para quê |
|---------|----------|
| `systemctl status nginx --no-pager` | Estado de um serviço sem abrir o paginador |
| `sudo systemctl disable --now nginx` | Para o serviço e impede que volte no boot |
| `pgrep -a ngrok` | Procura um processo pelo nome |

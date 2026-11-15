# Estudos de Redes

Repositório onde documento meus estudos teóricos sobre protocolos e comunicação em redes.

## Conteúdo

| Arquivo | Tema |
|---|---|
| [01 — Protocolo TCP/IP](docs/01-tcp-ip.md) | Modelo TCP/IP, camadas, endereçamento |
| [02 — Protocolo HTTP](docs/02-http.md) | HTTP, HTTPS, métodos, status codes |
| [03 — Requisições e Respostas HTTP](docs/03-requisicoes-respostas.md) | Estrutura de request e response, headers, body |
| [04 — Nginx exposto com ngrok](docs/04-subindo-servidor-nginx.md) | Servidor Nginx em VM Vagrant exposto na internet via ngrok e acessado pelo 5G |
| [05 — Arquitetura e Deploy](docs/05-arquitetura-e-deploy.md) | Containers, proxy reverso Traefik com Host Header, CI/CD e observabilidade |

## Estrutura do repositório

```
estudos-redes/
├── assets/img/
│   ├── http/       # imagens dos estudos de HTTP
│   └── nginx/      # evidências do lab Nginx + ngrok
├── docs/           # documentação teórica
└── README.md
```

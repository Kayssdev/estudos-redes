# Nginx exposto na internet com ngrok (Lab Vagrant)

Laboratório em que um servidor **Nginx** roda dentro de uma VM Linux (Vagrant + VirtualBox), é exposto na internet por um túnel **ngrok** e acessado por um celular na rede **5G**.

## Objetivo

Publicar um servidor web local (`localhost:80`) na nuvem e provar que ele é acessível de fora da rede de casa!

## Arquitetura

```
Celular (5G)
    │  https://<nome>.ngrok-free.dev
    ▼
Servidores do ngrok (nuvem)
    ▲
    │  túnel iniciado de DENTRO da VM (conexão de saída)
    │
ngrok (VM)  ──►  Nginx (VM, porta 80)
```

**Por que funciona:** o ngrok abre uma conexão de **saída** da VM até a nuvem dele. O celular acessa o endereço público do ngrok, e o tráfego volta pelo túnel até o Nginx.

**Por que não dá para usar o IP do PC no 5G:** o IP do PC é privado (`192.168.x.x`) e as operadoras móveis usam CGNAT, então conexões vindas de fora não chegam até a máquina.

## Ambiente

| Item  | Valor                             |
|-------|-----------------------------------|
| Host  | Windows com VirtualBox + Vagrant  |
| VM    | Linux baseado em Debian/Ubuntu    |
| Nginx | Instalado via `apt`               |
| ngrok | v3.39.13, plano Free              |

---

## 1. Instalar o Nginx

```bash
sudo apt update
sudo apt install -y nginx
sudo systemctl enable --now nginx
systemctl status nginx --no-pager
curl -I http://localhost
```

| Comando | O que faz |
|---------|-----------|
| `apt update` | Atualiza a lista de pacotes disponíveis. |
| `apt install -y nginx` | Instala o Nginx, respondendo "sim" automaticamente. |
| `systemctl enable --now nginx` | Inicia o serviço agora (`--now`) e faz ele subir junto com a VM (`enable`). |
| `systemctl status nginx` | Mostra se o serviço está ativo. |
| `curl -I http://localhost` | Pede só os cabeçalhos da resposta. `HTTP/1.1 200 OK` confirma que o servidor responde. |

![apt update e apt install nginx rodando na VM](../assets/img/nginx/03-instalacao-nginx.png)

## 2. Instalar o ngrok

Instalação via `snap`:

```bash
sudo snap install ngrok
```

O `snap` já resolve dependências e mantém o pacote atualizado automaticamente. Confirme a instalação:

```bash
ngrok version
```

> Alternativa: baixar o binário diretamente em `https://ngrok.com/downloads/linux`.

![snap install ngrok confirmando que o pacote já está instalado](../assets/img/nginx/04-instalacao-ngrok.png)

## 3. Autenticar o ngrok

1. Crie uma conta em <https://ngrok.com> (plano Free).
2. Copie o **authtoken** no dashboard.
3. Registre-o na VM:

```bash
ngrok config add-authtoken "token_do_usuário"
```

O token é salvo no arquivo de configuração do ngrok no home do usuário. **Nunca publique o token no GitHub.**

## 4. Personalizar a página inicial

O Nginx serve arquivos da pasta definida em `root` e escolhe o primeiro arquivo existente da lista `index`. Veja os dois valores:

```bash
sudo nginx -T | grep -E "root|index"
```

Saída neste lab:

```
root /var/www/html;
index index.html index.htm index.nginx-debian.html;
```

O Nginx percorre a lista da esquerda para a direita e serve o primeiro arquivo que existir. Neste lab, o `index.html` não estava presente na pasta, então o arquivo servido era o `index.nginx-debian.html`:

```bash
ls /var/www/html/
sudo nano /var/www/html/index.nginx-debian.html
```

Conteúdo usado neste lab:

```html
<!DOCTYPE html>
<html>
<head>
<title>Servidor Nginx exposto via Ngrok!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Pequeno exercicio feito para entender melhor os processos de servidor em nuvem, expondo o nginx via ngrok e acessando pelo 5G</h1>
<p>Exemplo simples para testes :D</p>

<p><em>Muito obrigado.</em></p>
</body>
</html>
```

- `color-scheme: light dark` respeita o tema do sistema operacional do visitante.
- `width: 35em; margin: 0 auto` centraliza o conteúdo.

Valide localmente (o `curl` não usa cache):

```bash
curl -s http://localhost
```

Arquivos estáticos não exigem reiniciar o Nginx.

![saída do nginx -T com root e index, e cat do index.nginx-debian.html](../assets/img/nginx/05-config-nginx-html.png)


## 5. Expor com o ngrok

```bash
ngrok http 80
```

A tela do ngrok mostra:

| Campo | Significado |
|-------|-------------|
| `Session Status` | `online` quando o túnel está ativo. |
| `Forwarding` | `https://<nome>.ngrok-free.dev -> http://localhost:80`: o endereço público e o destino local. |
| `HTTP Requests` | Lista das requisições recebidas (hora, método, caminho, status). |

Copie a URL de `Forwarding`.

![painel do ngrok mostrando Session Status online e requisições recebidas](../assets/img/nginx/02-painel-ngrok.png)

## 6. Testar pelo celular no 5G

1. Desligue o Wi-Fi do celular para garantir que o acesso é pela rede móvel.
2. Abra a URL do ngrok no navegador.
3. A página do Nginx deve carregar diretamente.

![celular acessando a página via URL ngrok-free.dev pelo 5G](/assets/img/nginx/01-acesso-celular-5g.png)

## 7. Evidências

Na VM, acompanhe o log ao vivo enquanto acessa pelo celular:

```bash
sudo tail -f /var/log/nginx/access.log
```

Resultado esperado:

```
127.0.0.1 - - [data] "GET / HTTP/1.1" 200 <bytes> "-" "Mozilla/5.0 (Linux; Android ..."
```

> O IP registrado é sempre `127.0.0.1` porque o tráfego chega à VM pelo túnel local do ngrok, não diretamente da internet. O IP real do celular fica visível apenas no painel do ngrok.

- Na tela do ngrok, as linhas `GET / 200 OK` confirmam que a requisição passou pelo túnel.

![tail -f do access.log registrando GET e HEAD do curl e do Android Firefox](../assets/img/nginx/06-access-log.png)

> O endereço **MAC** do celular não aparece em nenhum lugar: ele só existe dentro da rede local e é descartado no primeiro roteador.

## 8. Encerrar

Pressione `Ctrl+C` no terminal do ngrok. O endereço público deixa de funcionar imediatamente. Não deixe o túnel aberto depois da demonstração.


## Segurança

- Não publique o authtoken do ngrok, o `policy.yaml` com senha, nem prints com e-mail ou URL do túnel.
- Feche o túnel com `Ctrl+C` ao terminar.

## Referências

- Instalação do ngrok no Linux: <https://ngrok.com/downloads/linux>
- Documentação do Nginx: <https://nginx.org/en/docs/>
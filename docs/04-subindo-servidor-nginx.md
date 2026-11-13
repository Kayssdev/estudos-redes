# Nginx exposto na internet utilizando o **ngrok** (Lab Vagrant)

Laboratório em que um servidor Nginx roda dentro de uma VM Linux e é exposto na internet por um túnel ngrok e acessado por um celular na rede móvel.

Objetivo do Exercício

Publicar um servidor web local (localhost:80) na nuvem e provar que ele é acessível de fora da rede de casa.

Celular (5G)

    │  https://<nome>.ngrok-free.dev
    ▼

Servidores do ngrok (nuvem)

    ▲
    │  túnel iniciado de DENTRO da VM (conexão de saída)
    │

ngrok (VM)  ──►  Nginx (VM, porta 80)

**Como funciona:** O ngrok abre uma conexão de saída da VM  até a nuvem dele. O celular acessa o endereço público, e o tráfego volta pelo túnel até o Nginx.

**Por que não dá para usar o IP do PC no 5G:** O IP do PC é privado, e as operadoras utilizam CGNAT, então conexões de fora não chegam até a máquina!

## Ambiente utilizado

| Item  | Descritivo                        |
|-------|-----------------------------------|
| Host  | Windows com VirtualBox + Vagrant  |
| Nginx | Instalado via apt                 |
| ngrok | Plano Free                        |

|Comando               |	O que faz                           |
|----------------------|----------------------------|
| apt update           | 	Atualiza a lista de pacotes disponíveis. |
| apt install -y nginx |	Instala o Nginx, respondendo "sim" automaticamente. |
| systemctl enable --now nginx |	Inicia o serviço agora (--now) e faz ele subir junto com a VM (enable). |
| systemctl status nginx |	Mostra se o serviço está ativo. |
| curl -I http://localhost |	Pede só os cabeçalhos da resposta. HTTP/1.1 200 OK confirma que o servidor responde.
|




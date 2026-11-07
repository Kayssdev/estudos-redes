# Protocolo TCP/IP

## O que é TCP/IP?

Resumidamente o TCP/IP é a base fundamental que a maioria das redes foram construídas. Não é em si um único protocolo, porém uma suíte de protocolos que trabalham em conjunto!

## As camadas do modelo TCP/IP

### Aplicação(Interface)
O que faz: A Camada de Aplicação fornece serviços de rede diretamente às aplicações do usuário.
- HTTP, DNS, FTP, SMTP

### Transporte
O que faz: comunicação fim-a-fim entre aplicações, usando portas.
- TCP, UDP

### Rede
O que faz: endereçamento IP e roteamento entre redes diferentes.
- IP, ICMP
  
### Enlace
O que faz: transmite bits pelo meio físico; organiza dados em quadros (frames); usa endereço MAC.
- ARP, Ethernet, Wi-Fi

## IP — Endereçamento

IPv4: 32 bits, 4 octetos (192.168.33.10)

Privado (não roteado na internet, usa NAT): 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16.

Público: único no mundo, atribuído pelo provedor.

## TCP vs UDP

O que é TCP?

- O TCP (Transmission Control Protocol) é um protocolo da camada de transporte responsável por garantir a entrega confiável dos dados. Orientado a conexão e tem o fator Three-Way-Handshake, estabelecendo conexão entre o remetente e o destinatário

- UDP (User Datagram Protocol), que é mais rápido mas menos confiável. O UDP é utilizado em aplicações que priorizam velocidade ao contrário da confiabilidade, como streaming de vídeo e jogos online.

## Como os dados trafegam

De forma resumida o encapsulamento de dados funciona de uma forma simples, no exemplo a seguir

Aplicação: Mensagem/Dado é gerado.

Transporte: Porta de origem e destino. O resultado é o segmento

Rede: O segmento ganha um cabeçalho com IP de Origem e Destino. O resultado é o pacote.

Enlace: Adiciona o endereço MAC de origem e destino

A mensagem é enviada ao destinatário e é feito o processo inverso(Desencapsulamento).

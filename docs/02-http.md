# Protocolo HTTP

## O que é HTTP?

HTTP(Hypertext Transfer Protocol) é um protocolo de comunicação usado para a transferência de dados na World Wide Web(WWW). Ele é a base para qualquer troca de dados na web, e é um protocolo sem estado. Que significa que cada solicitação do cliente é tratada de forma independente pelo servidor, e sem reter informações sobre o estado da comunicação anterior.

## HTTP vs HTTPS

- HTTP: As informações trocadas entre o navegador do usuário e o servidor são transmitidas de forma não criptografada, o que significa que pode ocorrer interceptação de terceiros.
- HTTPS: Utiliza de uma camada adicional de segurança SSL/TLS para criptografar os dados entre o navegador e o servidor. Protege a integridade e confidencialidade de informações, tornando mais difícil a interceptação por terceiros.

Basicamente o "S" em HTTPS indica que a comunicação é segura e protegida.

## Métodos HTTP

- GET: Solicita dados de um recurso específico. É usado para recuperar informações do servidor.
  
- POST: Envia dados para serem processados a um recurso especificado. Usado para enviar informações para serem processadas para uma determinada URL.

- PUT: Atualiza um recurso existente ou cria um novo recurso se não existir. O cliente envia os dados do recurso com o local desejado especificado na URL.

- DELETE: Remove um recurso específico no servidor.

- OPTIONS: Retorna os métodos HTTP que o servidor suporta para um determinado URL. É útil para verificar as opções disponíveis para uma URL, como permissões de CORS (Cross-Origin Resource Sharing).

- PATCH: Aplica modificações parciais a um recurso. É usado para atualizar parcialmente um recurso existente.

- HEAD: Semelhante ao GET, mas solicita apenas os cabeçalhos do recurso, sem o corpo da resposta. É útil para obter informações sobre um recurso sem transferir todo o conteúdo.

## Códigos de status

- 200 OK: Solicitação bem-sucedida
- 201 Created: Recurso criado com sucesso.
- 204 No Content: Solicitação bem-sucedida, sem conteúdo a ser retornado
- 400 Bad Request: Requisição inválida do cliente.
- 401 Unauthorized: Falha na autenticação ou ausência de autorização.
- 403 Forbidden: O servidor entendeu a requisição, mas recusa o acesso.
- 404 Not Found: Recurso não encontrado.
- 500 Internal Server Error: Erro interno do servidor.

## Versões do HTTP

- HTTP/1.1, requisições processadas sequencialmente, utiliza-se o protocolo TCP
- HTTP/2, permite que múltiplas requisições possam ser enviadas simultaneamente por meio de uma única conexão TCP.
- HTTP/3, foi construído com base no protocolo QUIC, utiliza UDP em vez de TCP como camada de transporte.

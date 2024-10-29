# Configuração de um Broker MQTT com Mosquitto no Linux

Neste tutorial, você aprenderá a configurar um broker MQTT usando o software Open Source **Mosquitto** em uma máquina Linux na nuvem. Vamos usar o **Ubuntu 20.04** em uma instância **AWS EC2**, mas você pode adaptar as instruções para outras distribuições Linux e provedores de nuvem.

## Pré-requisitos
- Acesso administrativo (root) ao servidor Linux.

## Passo 1: Conecte-se ao Servidor

Conecte-se ao seu servidor usando SSH. Se você estiver usando uma instância EC2 no Ubuntu, execute no terminal:

```bash
ssh -i sua-chave.pem ubuntu@seu-ip-do-servidor
```

## Passo 2: Atualize os Pacotes do Sistema

Para garantir que os pacotes estejam atualizados, execute:

```bash
sudo apt-get update -y
sudo apt-get upgrade -y
```

## Passo 3: Instale o Mosquitto

O Mosquitto está disponível nos repositórios padrão do Ubuntu. Instale-o com:

```bash
sudo apt-get install mosquitto mosquitto-clients -y
```

## Passo 4: Configure o Mosquitto

Edite o arquivo de configuração em `/etc/mosquitto/mosquitto.conf`:

```bash
sudo nano /etc/mosquitto/mosquitto.conf
```

- Para alterar a porta, edite a linha:

  ```plaintext
  listener 1883
  ```

- Para habilitar o MQTT sobre Websockets, adicione:

  ```plaintext
  listener 9001
  protocol websockets
  ```

## Passo 5: Inicie e Ative o Mosquitto

Inicie o Mosquitto e configure-o para iniciar automaticamente com o sistema:

```bash
sudo systemctl start mosquitto
sudo systemctl enable mosquitto
```

## Passo 6: Abra as Portas do Servidor na Nuvem

Na AWS, abra as portas 1883 e 9001 no Security Group associado à sua instância EC2.

## Passo 7: Teste o Broker

- Em um terminal, inscreva-se em um tópico de teste:

  ```bash
  mosquitto_sub -h localhost -t test
  ```

- Em um segundo terminal, publique uma mensagem no tópico de teste:

  ```bash
  mosquitto_pub -h localhost -t test -m "hello world"
  ```

Você deve ver “hello world” no terminal onde executou o `mosquitto_sub`.

## Passo 8: Configuração de Autenticação de Usuário

1. Crie um arquivo de senha:

   ```bash
   sudo mosquitto_passwd -c /etc/mosquitto/passwd nome_usuario
   ```

2. No arquivo de configuração, adicione:

   ```plaintext
   allow_anonymous false
   password_file /etc/mosquitto/passwd
   ```

3. Ajustar Permissões do Arquivo de Senhas

Defina as permissões para que apenas o usuário root e o processo do Mosquitto possam acessar o arquivo:

```bash
sudo chmod 640 /etc/mosquitto/passwd
sudo chown root:mosquitto /etc/mosquitto/passwd
```


4. Reinicie o Mosquitto:

   ```bash
   sudo systemctl restart mosquitto
   ```

Para testar, forneça o nome de usuário e senha:

```bash
mosquitto_sub -h localhost -t test -u "nome_usuario" -P "senha"
mosquitto_pub -h localhost -t test -m "hello world" -u "nome_usuario" -P "senha"
```

## Configuração de Certificados TLS para Segurança

A seguir, vamos configurar a segurança usando certificados TLS para garantir a privacidade e integridade dos dados transmitidos.

### Passo 1: Instale o OpenSSL

Instale o OpenSSL para criar os certificados:

```bash
sudo apt-get install openssl -y
```

### Passo 2: Crie a Autoridade Certificadora (CA)

```bash
mkdir ~/certs
cd ~/certs
openssl genrsa -out ca.key 2048
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt
```

### Passo 3: Crie os Certificados do Servidor

1. Crie a chave privada do servidor:

   ```bash
   openssl genrsa -out server.key 2048
   ```

2. Crie a solicitação de assinatura de certificado (CSR):

   ```bash
   openssl req -new -key server.key -out server.csr
   ```

3. Assine a CSR com a chave da CA:

   ```bash
   openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 3650
   ```

### Passo 4: Crie os Certificados do Cliente

1. Crie a chave privada do cliente e o CSR:

   ```bash
   openssl genrsa -out client.key 2048
   openssl req -new -key client.key -out client.csr
   ```

2. Assine a CSR do cliente:

   ```bash
   openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 3650
   ```

### Passo 5: Configure o Mosquitto para TLS

No arquivo de configuração do Mosquitto, adicione:

```plaintext
listener 8883
cafile /home/ubuntu/certs/ca.crt
certfile /home/ubuntu/certs/server.crt
keyfile /home/ubuntu/certs/server.key
require_certificate true
```

Reinicie o Mosquitto:

```bash
sudo systemctl restart mosquitto
```

### Passo 6: Teste a Configuração com TLS

Teste com:

```bash
mosquitto_pub -h localhost -p 8883 --cafile ~/certs/ca.crt --cert ~/certs/client.crt --key ~/certs/client.key -t test -m "hello tls" -d
mosquitto_sub -h localhost -p 8883 --cafile ~/certs/ca.crt --cert ~/certs/client.crt --key ~/certs/client.key -t test -d
```

---

Com isso, seu broker MQTT está configurado com TLS. Para uma configuração de produção, recomenda-se o uso de certificados individuais para cada cliente e a revogação de certificados quando necessário.
# Cloudflare Mikrotik

Aqui vamos criar 2 containers para usar na cloudflare

# cloudflared



# SSH Server

## Tipo de chaves e certificados

Cloudflare usa chave curta assinada do tipo ecdsa-sha2-nistp256 se precisar criar uma igual para testar se o servidor esta funcionando use o comando a baixo.

CA
```
ssh-keygen -t ecdsa -b 256 -f ca_key
```
Usuário user
```
ssh-keygen -t ecdsa -b 256 -f user_key
ssh-keygen -s ca_key -I user -n user -V +52w user_key.pub
```
```
ssh -i user_key -o CertificateFile=user_key-cert.pub user@servidor

```
Usuário joao
```
ssh-keygen -t ecdsa -b 256 -f joao_key
ssh-keygen -s ca_key -I joao -n joao -V +52w joao_key.pub
```
```
ssh -i usuario_key -o CertificateFile=usuario_key-cert.pub user@servidor

```

Objetivo é acessar SSH através da Cloudflare.

### Gerar a chave SSH

Eu acho que é desnecessário re-escrever essa explicação veja o link
[Generate a Cloudflare SSH CA](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/use-cases/ssh/ssh-infrastructure-access/#generate-a-cloudflare-ssh-ca)

1) Crie a chave API
2) Crie o certificado
3) Salve em um arquivo como ca.pub

ca.pub

4) Coloque o arquivo no servidor
5) Edite a permissão do arquivo

```
chmod 600 /etc/ssh/ca.pub
```

### Configurar o servidor ssh para usar o certificado

Os passos a seguir também estão no link do manual da cloudflare mas precisam ser feitos no servidor.

1) No servidor edite /etc/ssh/sshd_config
```
PubkeyAuthentication yes
TrustedUserCAKeys /etc/ssh/ca.pub
```
2) Reiniciar o servidor SSH


### Fingerprint

No log só aparece o fingeprint do certificado que foi usado, para comparar use o comando a baixo para poder ver o do seu certificado.

```
ssh-keygen -lf /etc/ssh/ca.pub
```
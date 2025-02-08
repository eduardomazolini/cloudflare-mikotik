# SSH Server

A ideia aqui é criar um **Servidor SSH** que aceite login de **qualquer usuário** que tenha um **certificado** confiável.

Nessa implementação não vamos criar todos os usuários no servidor.  
Se fosse eles deveriam ser o o que tiver antes do **@** do e-mail das pessoas.

Então vamos escolher um usuário e todos vão acessar suas pastas e ter suas permissões.
Para isso tive a sorte de encontrar a biblioteca [Name Service Switch module All-To-One (libnss-ato)](https://github.com/donapieppo/libnss-ato)

## Dicas

### Tipo de chaves e certificados

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
ssh -i user_key -o CertificateFile=user_key-cert.pub user@servidor

```

### Fingerprint

No log só aparece o fingeprint do certificado que foi usado, para comparar use o comando a baixo para poder ver o do seu certificado.

```
ssh-keygen -lf /etc/ssh/ca.pub
```
Objetivo é acessar SSH através da Cloudflare.

## Montando o Servidor


### Escolher a distro Linux

Sempre que possível minha escolha pessoal é pelo Alpine Linux, mas não é possível. Alpine usa o [musl](https://musl.libc.org/) e não é compatível com o [**Name Service Switch module All-To-One**](https://github.com/donapieppo/libnss-ato/issues/19)

Neste caso eu precisei usar [Debian](https://hub.docker.com/_/debian) e pra manter pequeno escolhi SLIM, especificamente TAG 12-slim.


### Instalar o serviço

Parece obvio mas containers tem o mínimo de coisas instaladas.
Então precisamos instalar **openSSH Server**.
Pra tentar mater pequeno vamos evitar de instalar o que é recomendado por padrão.

```
apt install -y --no-install-recommends openssh-server
```

### Configuração inicial

Os comandos a baixo são essenciais para roda o servidor.

```
ssh-keygen -A
```
O servidor também precisa ter chaves para fazer a criptografia do lado dele. O comando a cima cria essas chaves. 
O Debian já executou esse comando na Instalação. O Alpine não, então tem que criar antes de roda a primeira vez.

```
mkdir -p /run/sshd
```
Apine não precisa criar o diretório no Debian precisa. Nele é criado o arquivo com o PID do processo.

### Gerar a chave SSH CA

A Documentação oficial da Cloudflare hoje chama o acesso com o comando ``cloudflared access`` de **Legacy**
Mas o acesso **browser-based** usa também esse certificado e não o **~~Infrastructure Access~~** e descobrir isso foi uma das coisas que me tomou mais tempo. 

Assim recomendo que envie os **2 certificados** pro seu servidor.  
Coloque uma chave publica em cada linha do **ca.pub**


#### Chave SSH Infrastructure Access
Eu acho que é desnecessário re-escrever essa explicação veja o link
[Generate a Cloudflare SSH CA](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/use-cases/ssh/ssh-infrastructure-access/#generate-a-cloudflare-ssh-ca)

1) Crie a chave API
2) Crie o certificado
3) Salve em um arquivo como ca.pub

#### Chave SSH Short Lived Certificates

Antes de baixa o certificado é necessário criar a [Aplicação do tipo Self-Hosted tipo **BROWSER SSH**](https://developers.cloudflare.com/cloudflare-one/applications/non-http/browser-rendering/) e escolher para ele renderizar o terminal SSH.

Os detalhes também estão na [Documentação Oficial](https://developers.cloudflare.com/cloudflare-one/applications/non-http/short-lived-certificates-legacy/#3-generate-a-short-lived-certificate-public-key)

No Zero Trust, acesse  **Access > Autorização de Serviço > SSH**
Escolha a aplicação e clique em **Gerar Certificado**

4) Copie a chave e cole na linha seguinte do ca.pub.

#### Novamente no servidor

5) Coloque o arquivo no servidor em:
```
/etc/ssh/ca.pub
```
6) Edite a permissão do arquivo:
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

### Principals

> Não precisei usar. Pode ignorar esse trecho.

Os certificados tem um informação que é cruzada com o usuário.
Assim o certificado do "joão" não acessa o usuário da "maria".
Mas o certificado pode ser do "Dep de Suporte" e eles podem acessar o "root".
Neste caso que o principals não é o usuário é preciso faze uma configuração no servidor.

```
AuthorizedPrincipalsFile /etc/ssh/auth_principals/%u
```

```
AuthorizedPrincipalsCommandUser root
AuthorizedPrincipalsCommand /opt/local/bin/authorized-principals.sh %i
```
Lembre de caso for fazer o script iniciar ele com:
```
#!/bin/sh
```

### Criando o Usuário

Meu objetivo é que todos entrem com mesmo usuário.
Então eu vou criar aqui um usuário **"guest"** e todos vão usar ele.

```
useradd -m -d /home/guest -s /bin/bash guest
```

# [Name Service Switch module All-To-One (libnss-ato)](https://github.com/donapieppo/libnss-ato)

Essa foi a grande descoberta que nem as inteligências artificiais conseguiram me dar a dica, penei muito para achar.
Quero agradecer muito ao desenvolvedor [Donapieppo](https://github.com/donapieppo) por ter feito essa lib.

Por favor, vão até o github e marquem o repositório com ⭐
https://github.com/donapieppo/libnss-ato

## Compilando

```
wget -O libnss-ato.zip https://github.com/donapieppo/libnss-ato/archive/refs/heads/master.zip

unzip libnss-ato.zip
cd libnss-ato-master

apt install gcc

```

Eu não to instalando o MAKE pq o comando do gcc é simple e no momento o código tem um WARNING que o comando gerado pelo MAKE não ignora então teria que colocar um -w no final e rodar o comando diretamente.

```
gcc -fPIC -Wall -shared -o libnss_ato.so.2 -Wl,-soname,libnss_ato.so.2 libnss_ato.c -w
```
## Configurando

Eu fiz todo essa compilação em um docker especifico e salvei o arquivo pra depois usar ele, salvando na pasta correta.

```
cp libnss_ato.so.2 /lib/
```
Então rode o comando ``ldconfig`` para atualiza o cache de bibliotecas compartilhadas. 

Configure o arquivo do NSS e no meu caso eu poderia zerar tudo e deixar só as configurações para essa biblioteca.  
Edite o ``/etc/nsswitch.conf`` como abaixo.

```
#/etc/nsswitch.conf
passwd:         files ato
group:          files
shadow:         files ato
```

Por fim vamos dizer qual usuário será usado.  
Para isso copie a linha do `/etc/passwd` referente ao usuário no `/etc/libnss-ato.conf`.

```
cat /etc/passwd | grep guest > /etc/libnss-ato.conf
```
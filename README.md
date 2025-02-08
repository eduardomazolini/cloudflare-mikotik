# Container para mikrotik

## Ambiente Docker

**Opcional:** Siga esses instruções se pretende criar os container usando seu PC.

### Instalação do Docker
Segue link de como instalar [Docker Engine][docker_install] no seu PC.

### Instalar build para as plataformas ARM, ARM4 e outras
No [documentação da mikrotik][mikrotik] tinha uma dica muito boa de como criar o container para arquiteturas diferentes.

```
docker run --privileged --rm tonistiigi/binfmt --install all
```

### Logar no Docker Hub para enviar sua imagem
Para logar sem problemas e poder enviar sua imagem, você não pode user o comando sudo então habilite seu usuário para usar docker.

```
sudo usermod -aG docker $USER 
newgrp docker
docker login

```

## Configurações do Mikrotik

### Instalar o pacote opcional container
No site da [Mikrotik -> Software][mk_software] escolha sua arquitetura e baixe o **Extra packages** ele é um zip, descompacte e suba o pacote no mikrotik depois reinicie.
Eu recomendo instalar o mínimo de pacotes para ter mais espaço livre.

[mk_software]: https://mikrotik.com/download

### Ativar o uso de containers
Não basta instalar o pacote.
Por segurança também é necessário ativer o uso de container, para isso deve efetuar o comando e pressionar o botão físico de reset.

fonte: https://help.mikrotik.com/docs/spaces/ROS/pages/84901929/Container#Container-EnableContainermode

```
/system/device-mode/update container=yes
```

### Baixando containers

**Opcional:** Se for instalar o container comm o envio do arquivo **.tar**

Para poder baixar as imagens informe o local de onde as imagens estão, normalmente o [Docker Hub][docker_hub]
Se suas imagens são privadas então informe seu usuário e senha do [Docker Hub][docker_hub]

**ATENÇÃO:** Sua **senha** ficara **visível** ao comando print, pense em criar um usuário dedicado com acesso aos seus arquivos.

**ATENÇÃO:** Se sua RB tiver opção de adicionar um disco externo faça isso e configure o tmpdir.

```
/container config
set registry-url=https://registry-1.docker.io \
 tmpdir=pull \
 username=usuario \
 password=XXXXXXX
```

### Preparando a rede 

Cada container como um PC deve ter pelo menos uma interface de rede e um IP, exceto em casos muito específicos nos PCs, já no mikrotik é obrigatório informar uma interface virtual.

#### Virtual Ethernet

No comando a baixo iremos criar a interface de rede do container e atribuir o IP e gateway que ele ira utilizar.
Como exemplo o container ficou com o ip final 5 e o gateway eu considerei final 1.

```
/interface veth
add address=192.168.10.2/30 gateway=192.168.10.1 name=veth1
```

#### Preparando o gateway

O container pode ser colocado em uma bridge com outra porta ligada a LAN e assim participar da rede como qualquer outro host com IP fixo e assim usar o gateway desta rede.

Supondo que a RB que ele está hospedado será o gatway e ele ira ficar isolado, vamos seguir.

```
/interface/bridge/add name=containers
/ip/address/add address=192.168.10.1/30 interface=containers
/interface/bridge/port add bridge=containers interface=veth1
```

Supondo que por ser isolado o resto da rede não o conhece irei fazer o NAT

```
/ip firewall nat
add chain=srcnat action=masquerade src-address=192.168.10.0/30
```

## Container dicas e truques

### Container de Test

Alpine é uma imagem bem leve que pode ser usada para testes.

```
/container
add remote-image=alpine interface=veth1 cmd="/dev/null" entrypoint="tail -f"
```

### Variáveis de ambiente

O modo mais elegante de configurar um bom container.
Para isso crie uma lista de variáveis e atribua ao container.

Ex: o nível de detalhes de um log.
```
/container envs
add key=TUNNEL_LOGLEVEL name=cloud value=debug

/container
set 0 envlist=cloud 
```

### Disco e Arquivos

Um container deve ser descartável, assim sendo se ele precisar salvar alguma informação em arquivo ou ler um arquivo customizado este deve ser salvo no seus sistema de arquivo externo e mapeado para dentro do container.
Pode ser um arquivo só ou toda uma pasta.
Para isso crie uma lista de volumes e atribua ao container.

Ex: O arquivo de certificado SSH

```
/container mounts
add dst=/etc/ssh/ca.pub name=SSH src=/ca.pub

/container
set 0 mounts=CA
```

### Substituir o ENTRYPOINT e CMD
Quando o container é iniciado ele segue 2 caminhos de inicialização

1) Shell
Se o entrypoint não foi especificado entre [] na criação.
O SHELL é implicitamente é: ```["/bin/sh", "-c"]``` 
Será executado SHELL seguido como argumento o ENTRYPOINT e o CMD

```
ENTRYPOINT echo OLA
CMD Mundo
```

2) Exec (recomendado)
Quando o entrypoint foi especificado entre [] na criação.
será executado diretamente o ENTRYPOINT seguido do CMD como argumento.

Neste caso ira dar erro:
```
ENTRYPOINT ["set"]
CMD ["Mundo"]
```
Mensagem de erro: ```execve: No such file or directory```

Para funcionar use:
```
ENTRYPOINT ["/bin/sh -c set"]
CMD ["Mundo"]
```

Por linha de comando  em um PC o CMD é facilmente substituído passando argumento após o nome da imagem.

### Ver as variáveis

Substitua o entrypoint
```
/container
print
set 0 entrypoint="sh -c set"
```

### Manter a maquina rodando para acessar o shell

```
/container
print
set 0 set 0 cmd="/dev/null" entrypoint="tail -f"
```

### Log

Ao rodarmos um serviço os logs gerados podem ser visos no MikroTik usando parâmetro logging.

```
/container
print
set 0 set 0 logging=yes
```

Você pode definir onde ele será visível usando tópico container

```
/system logging
add action=echo topics=container
```
**Atenção:** No caso de ```echo``` se tiver muitas mensagens elas podem ser ocultadas no terminal exibindo ```(X messages discarded)``` então confira o log de outra forma.

### Inicia ao reiniciar a RB
Ao rodarmos um serviço na inicialização para que ele volte a subir depois de religar o MikroTik use o parâmetro start-on-boot.

```
/container
print
set 0 set 0 start-on-boot=yes
```

## [Cloudflare Tunnel][cf_tunel_readme]

Túnel cloudflared permite acesso da internet a sua rede através de aplicativos e servidor DNS

### Criando a imagem

```
cd cloudflared
docker buildx build  --no-cache --platform arm --output=type=docker -t eduardomazolini/cloudlared-arm-mikrotik .

docker save eduardomazolini/cloudlared-arm-mikrotik > cloudlared-arm-mikrotik.tar

docker push eduardomazolini/cloudlared-arm-mikrotik:latest
```

### No MikroTik

Lembre de configurar a rede, a interface virtual e configurar o Docker Hub para baixara imagem.

```
/container envs
add key=TUNNEL_TOKEN name=cloud value=eyXXXXXXXXXXXXX
add key=TUNNEL_LOGLEVEL name=cloud value=debug
```

```
/container
add envlist=cloud interface=veth1 logging=yes remote-image="eduardomazolini/cloudlared-arm-mikrotik" start-on-boot=yes
```

#### Túnel com configuração Local
Se for criar um túnel configurado localmente lembre de mapear **/root/.cloudflared**

Crie a pasta e se já tiver os arquivos faça upload
```
/file
add type=directory name=cloudflared
```

Crie o mapeamento
```
/container mounts
add dst=/root/.cloudflared/ name=cloud src=/cloudflared
```
Adicione o mapeamento no container.
```
/container
set 0 mounts=cloud
```


## [SSH Server][ssh_readme]
Um servidor que loga sem pedir senha ou chave apenas com autenticação da Cloudflare.

Muito útil para administração remota.

```
cd cloudflared
docker buildx build  --no-cache --platform arm --output=type=docker -t eduardomazolini/cloudlared-arm-mikrotik .

docker save eduardomazolini/cloudlared-arm-mikrotik > cloudlared-arm-mikrotik.tar

docker push eduardomazolini/cloudlared-arm-mikrotik:latest
```




[docker_install]: https://docs.docker.com/engine/install/

[mikrotik]:https://help.mikrotik.com/docs/spaces/ROS/pages/84901929/Container#Container-c\)buildanimageonPC

[cf_tunel_readme]: cloudflared/README.md

[ssh_readme]: ssh-mk/README.md

[docker_hub]: [https://hub.docker.com]
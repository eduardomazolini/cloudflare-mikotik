# Clodflared Tunnel

Esse projeto tem como objetivo criar container compatível com MikroTik ARM (arm32).

## Alpine

[![Logo Alpine Linux][logo_alpine]](https://alpinelinux.org/ 'Alpine Linux official website')

Small. Simple. Secure.
Alpine Linux is a security-oriented, lightweight Linux distribution based on musl libc and busybox.

### APK (Alpine Package Keeper)

[Upgrade the system][apk_upgrade]

```
apk update
apk upgrade
```

### Dpkg (Debian *p*ac*k*a*g*e) (Opcional)

_Existe a opção de baixar o binário, tornando desnecessário o instalador._

O cloudflared túnel é distribuído como pacote debian (.deb) , para instalar precisamos do dpkg
```
apk add dpkg
```

O pacote precisa da arquitetura com o nome exato "arm", verifique se existe.
```
dpkg --print-architecture 
dpkg --print-foreign-architectures
```

Adicione o nome da arquitetura "arm" e confira se aparece na lista agora.
```
dpkg --add-architecture arm
dpkg --print-foreign-architectures
```


## Cloudflared Tunnel Download e Instalação

Na [documentação oficial da cloudflare][cf_down_doc] pode ser encontrado o link que usaremos.
Atenção que normalmente eles indicam curl que não esta na instalação padrão do Alpine, usaremos wget.

### Pacote deb
Após o download já rodamos a instalação

```
wget -O cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm.deb

dpkg -i cloudflared.deb
```

---

### Binário

```
wget -O cloudflared https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm

```

### Crie o túnel

Siga as instruções do da [Documentação da CloudFlare][cf_create_tunnel]
Anote 2 informações importantes

1) **Token** que foi parcialmente coberto nessa imagem
![Token](https://developers.cloudflare.com/_astro/connector.DgDJjokf_1bYl1O.webp)
Ele é usado na variável de ambiente TUNNEL_TOKEN para iniciar o túnel que é remotamente administrado.

2) **Tunnel ID** Ele é um UUID, usado no DNS para direcionar [nome de domínio público][cf_cname] usando um registro CNAME com target \<UUID\>.cfargotunnel.com.

    **Atenção:** Não confunda **Tunnel ID** com **Connector ID**.

### Executar

Em containers não deve existir diferentes serviços rodando por isso se vc executar o comando sugerido para registrar o serviço do clouflared não ira funcionar.
Recomendo criar uma variável de ambiente com o token do túnel

~~sudo cloudflared service install eyxxxxxxxxx~~

```
cloudflared tunnel run --token $TUNNEL_TOKEN
```
Para maiores detalhes dos parâmetros veja a [documentação][cf_tunel_doc]



[logo_alpine]: https://alpinelinux.org/alpinelinux-logo.svg
[apk_upgrade]: https://wiki.alpinelinux.org/wiki/Alpine_Package_Keeper#Upgrade_a_Running_System
[cf_down_doc]: https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/
[cf_tunel_doc]: https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/configure-tunnels/tunnel-run-parameters/
[cf_create_tunnel]: https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/get-started/create-remote-tunnel/
[cf_cname]: https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/routing-to-tunnel/dns/#create-a-dns-record-for-the-tunnel
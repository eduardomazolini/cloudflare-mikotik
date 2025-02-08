# Primeiro estágio: Compilação
FROM debian:12-slim AS builder

WORKDIR /app

# Instala dependências de compilação
RUN apt update \
&& apt install -y wget unzip gcc

# Baixa e compila libnss-ato
RUN wget -O libnss-ato.zip https://github.com/donapieppo/libnss-ato/archive/refs/heads/master.zip \
    && unzip libnss-ato.zip \
    && cd libnss-ato-master/ \
    && gcc -fPIC -Wall -shared -o libnss_ato.so.2 -Wl,-soname,libnss_ato.so.2 libnss_ato.c -w

# Segundo estágio: Imagem final
FROM debian:12-slim

WORKDIR /app

# Copia a biblioteca compilada do primeiro estágio
COPY --from=builder /app/libnss-ato-master/libnss_ato.so.2 /lib/

# Configura ldconfig
RUN ldconfig

# Cria o usuário guest e configura libnss
RUN useradd -m -d /home/guest -s /bin/bash guest \
    && cat /etc/passwd | grep guest > /etc/libnss-ato.conf

# Limpeza (opcional, mas recomendado para reduzir ainda mais o tamanho)
RUN apt update && \
    apt install -y --no-install-recommends openssh-server && \
    rm -rf /var/lib/apt/lists/*

# Não confere a senha no PAM
RUN sed -i "1i auth sufficient pam_permit.so" /etc/pam.d/sshd

RUN echo "passwd:         files ato"> /etc/nsswitch.conf && \
    echo "group:          files">> /etc/nsswitch.conf && \
    echo "shadow:         files ato">> /etc/nsswitch.conf

RUN mkdir /var/run/sshd && \
    echo "PermitRootLogin no" >> /etc/ssh/sshd_config && \
    echo "PasswordAuthentication no" >> /etc/ssh/sshd_config && \
    echo "PubkeyAuthentication yes" >> /etc/ssh/sshd_config && \
    echo "PermitEmptyPasswords no" >> /etc/ssh/sshd_config && \
    echo "TrustedUserCAKeys /etc/ssh/ca.pub" >> /etc/ssh/sshd_config

# Gerar as chaves de host SSH
RUN ssh-keygen -A

# Cria a pasta run
RUN mkdir -p /run/sshd

# Copiar chave CA pública para o container (ajuste conforme necessário)
COPY cloudflare/ca.pub /etc/ssh/ca.pub
RUN chmod 600 /etc/ssh/ca.pub

# Configurar portas e iniciar o servidor SSH
EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]

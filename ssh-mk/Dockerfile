FROM arm32v7/alpine:latest

# Instalar OpenSSH e ferramentas necessárias
RUN apk --no-cache add openssh && \
    mkdir /var/run/sshd && \
    echo "PermitRootLogin no" >> /etc/ssh/sshd_config && \
    echo "PasswordAuthentication no" >> /etc/ssh/sshd_config && \
    echo "PubkeyAuthentication yes" >> /etc/ssh/sshd_config && \
    echo "PermitEmptyPasswords no" >> /etc/ssh/sshd_config && \
    echo "TrustedUserCAKeys /etc/ssh/ca.pub" >> /etc/ssh/sshd_config

# Gerar as chaves de host SSH
RUN ssh-keygen -A

# Adicionar usuário para SSH (substitua USERNAME pelo nome desejado)
RUN adduser -D -s /bin/sh user && \
    mkdir -p /home/user/.ssh && \
    chown -R user:user /home/user/.ssh

# Copiar chave pública para o container (ajuste conforme necessário)
COPY your-public-key.pub /home/user/.ssh/authorized_keys
RUN chown user:user /home/user/.ssh/authorized_keys && \
    chmod 600 /home/user/.ssh/authorized_keys

# Configurar portas e iniciar o servidor SSH
EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]

# 📘 Manual de Instalação e Configuração do WireGuard (VPN)
## 🔎 O que é o WireGuard?

WireGuard é uma VPN moderna, simples e rápida, que cria uma rede privada criptografada. Com ele, todos os dispositivos conectados (servidor e clientes) se enxergam como se estivessem na mesma rede local.


## 🖥️ 1. Instalação no Servidor (Linux)
### A. Requisitos

- **Uma máquina Linux (Ubuntu/Debian/Fedora/etc.).**

- **Pode ser uma VM na AWS (EC2) ou uma VM local no VirtualBox.**

- **Acesso root ou sudo.**

- **Porta 51820/UDP liberada no firewall e no roteador/Security Group.**

### B. Instalar WireGuard
```bash
sudo apt update && sudo apt install wireguard -y
```

**Para Fedora/CentOS:**
```bash
sudo dnf install wireguard-tools -y
```

### C. Gerar chaves e mudar permissões
Utilize o próximo comando para criar uma chave privada condificada em *base64* que ficará armazenada apenas no servidor e em seguida altere as permissões para que nenhum outro usuário veja:
```bash
wg genkey | sudo tee /etc/wireguard/private.key
sudo chmod go= /etc/wireguard/private.key
```
- **private.key → chave privada (fica no servidor).**

O próximo comando gerará uma chave pública também em *base64* a partir da chave privada, tenha uma cópia dela para que possa distribuir para as máquinas que acessarão a VPN:
```bash
sudo cat /etc/wireguard/private.key | wg pubkey | sudo tee /etc/wireguard/public.key
```
- **public.key → chave pública (vai para os clientes).**


## 🖥️ 2. Configuração do Servidor

Edite o arquivo */etc/wireguard/wg0.conf*, crie o arquivo e o diretório caso não exista:
```ini
[Interface]
Address = 10.8.0.1/24
PrivateKey = (chave_privada_servidor)
ListenPort = 51820
SaveConfig = True

# Exemplo de cliente
[Peer]
PublicKey = (chave_publica_cliente1)
AllowedIPs = 10.8.0.2/32
```
Nesta configuração você pode escolher qualquer intervalo de endereço IPv4 privado como:
- **10.0.0.0** to **10.255.255.255** (10/8 prefix)
- **172.16.0.0** to **172.31.255.255** (172.16/12 prefix)
- **192.168.0.0** to **192.168.255.255** (192.168/16 prefix)

Lembrando que para cada máquina cliente que for se conectar na VPN você precisa criar um **[Peer]** colocando a chave pública correspondente e atribuindo o IP diferente do que já foi atribuído. 

Quanto a opção **SaveConfig** garante que quando a interface do WireGuard for desligada, qualquer configuração salva será armazenada no arquivo de configuração.

Ativar a interface:
```bash
sudo wg-quick up wg0
```

Habilitar no boot:
```bash
sudo systemctl enable wg-quick@wg0
```

Liberar firewall (Ubuntu/UFW):
```bash
sudo ufw allow 51820/udp
sudo ufw allow in on wg0
sudo ufw allow out on wg0
```
- **caso precise saber sobre mais configurações de firewall ou IPv6 consulte o link para o manual do site DigitalOcean ao final deste manual.**

## 💻 3. Instalação no Cliente (Peer)
###  A. Linux
```bash
sudo apt update && sudo apt install wireguard -y
```

Gerar chaves e gerenciar permissões:
```bash
# chave privada
wg genkey | sudo tee /etc/wireguard/private.key
sudo chmod go= /etc/wireguard/private.key

# chave pública
sudo cat /etc/wireguard/private.key | wg pubkey | sudo tee /etc/wireguard/public.key
```


Criando Arquivo */etc/wireguard/wg0.conf*:
```ini
[Interface]
Address = 10.8.0.2/24
PrivateKey = (chave_privada_cliente)
# DNS = 1.1.1.1

[Peer]
PublicKey = (chave_publica_servidor)
Endpoint = IP_PUBLICO_SERVIDOR:51820
AllowedIPs = 10.8.0.0/24
PersistentKeepalive = 25
```
Nessa configuração tenha atenção ao IP, você deve colocar na [Interface] o IP que foi configurado no servidor para a máquina cliente. A configuração de DNS é para caso queira fazer com que sua navegação saia pela VPN.

Subir a conexão:
```bash
sudo wg-quick up wg0
```

### B. Windows

1. Baixe o WireGuard oficial

2. Gere as chaves pelo próprio aplicativo (ou importe já prontas).

3. Clique em Add Tunnel → Add empty tunnel….

4. Copie a configuração client.conf.

5. Clique em Activate para iniciar a VPN.

### C. macOS

1. Instale pelo App Store

2. Importe ou crie um túnel.

3. Cole a configuração client.conf.

4. Ative o túnel.

## 🔗 4. Comunicação entre Servidor e Cliente

- No servidor, veja o status:
```bash
sudo wg
```
➡️ Deve mostrar latest handshake e os peers conectados.

- No cliente:
```bash
sudo wg
```
➡️ Deve mostrar peer com o servidor e bytes transferidos.


- Teste conectividade:
```bash
ping 10.8.0.1   # Do cliente para o servidor
ping 10.8.0.2   # Do servidor para o cliente
```

## 🌐 5. AWS (caso use EC2)

### Além dos passos acima, é obrigatório:

1. No Security Group da instância, abrir a porta 51820/UDP para o mundo (0.0.0.0/0).

2. (Opcional para segurança) restringir a porta UDP só para os IPs dos clientes conhecidos.

3. A VM terá tanto o IP público AWS (para o Endpoint) quanto o IP privado WireGuard (10.8.0.1).

4. Para logar via SSH com segurança, use o IP da VPN (ssh usuario@10.8.0.1).

## 🖥️ 6. VirtualBox (caso use localmente)

- Configure a VM Linux como servidor WireGuard normalmente.

- No host, use o cliente (Windows/Linux/macOS).

- O tráfego passa pela rede do host, mas dentro da VPN vai criptografado.

- Pode usar NAT ou Bridge no VirtualBox sem problemas, contanto que a porta 51820/UDP esteja acessível.

## ✅ 7. Checklist de Conexão

- Chaves corretas (privada no dispositivo, pública no peer remoto).

- Servidor com IP 10.8.0.1/24.

- Cliente com IP 10.8.0.2/24.

- Porta 51820/UDP aberta no firewall/roteador/AWS.

- AllowedIPs → /32 no servidor, /24 no cliente.

- PersistentKeepalive = 25 no cliente.

- Handshake aparece em **wg show**.

- Pings funcionam entre os IPs da VPN.

## 📌 Resumo:

- Servidor = chave privada + IP 10.8.0.1 + ListenPort 51820.

- Cliente = chave privada + IP 10.8.0.2 + aponta para IP público do servidor.

- Handshake precisa aparecer em ambos (wg show).

- Uma vez ativo, tudo o que passa pela rede 10.8.0.0/24 está criptografado.

## Referências
- [tutorial DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-set-up-wireguard-on-ubuntu-20-04)
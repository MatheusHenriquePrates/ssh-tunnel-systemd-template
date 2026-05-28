<p align="center">
  <a href="#-portugu%C3%AAs"><img src="https://img.shields.io/badge/-PT--BR-39d353?style=for-the-badge&labelColor=0d1117" alt="PT-BR"/></a>
  &nbsp;
  <a href="#-english"><img src="https://img.shields.io/badge/-EN-58a6ff?style=for-the-badge&labelColor=0d1117" alt="EN"/></a>
</p>

<p align="center">
  <img src="https://capsule-render.vercel.app/api?type=slice&color=0:0d1117,100:39d353&height=180&section=header&text=ssh-tunnel-systemd-template&fontSize=30&fontColor=ffffff&animation=fadeIn&fontAlignY=42&desc=t%C3%BAnel%20ssh%20que%20de%20fato%20sobrevive%20%C3%A0%20falha%20%C2%B7%20via%20systemd&descAlignY=68&descSize=14" width="100%" />
</p>

<p align="center">
  <img src="https://readme-typing-svg.demolab.com/?lines=%24+matheus%40devops%3A~%24+ssh-tunnel-systemd-template;%24+sem+aquele+estado+zumbi+%22ssh+vivo%2C+forward+morto%22;%24+ExitOnForwardFailure%2C+ServerAlive%2C+Requires+wg;%24+ou+a+conex%C3%A3o+flui%2C+ou+systemd+est%C3%A1+tentando&font=Fira%20Code&size=18&pause=1200&color=39D353&center=true&vCenter=true&width=720&height=45" />
</p>

<a id="-português"></a>

## PT-BR

```bash
matheus@devops:~$ cat sobre.txt
```

Unit template do systemd pra **túneis SSH que de fato sobrevivem à falha** — sem cair no estado zumbi "processo ssh existe, mas o forward tá morto" que a maioria dos tutoriais produz.

```bash
matheus@devops:~$ ls stack/
```

![SSH](https://img.shields.io/badge/-SSH-0d1117?style=for-the-badge&logo=openssh&logoColor=39d353) ![systemd](https://img.shields.io/badge/-systemd-0d1117?style=for-the-badge&logo=systemd&logoColor=39d353) ![Linux](https://img.shields.io/badge/-Linux-0d1117?style=for-the-badge&logo=linux&logoColor=39d353) ![WireGuard](https://img.shields.io/badge/-WireGuard-0d1117?style=for-the-badge&logo=wireguard&logoColor=39d353) ![Bash](https://img.shields.io/badge/-Bash-0d1117?style=for-the-badge&logo=gnubash&logoColor=39d353)

```bash
matheus@devops:~$ cat por-que-existe.txt
```

Uma unit naïve do systemd pra túnel SSH é tipo isso:

```ini
[Service]
ExecStart=/usr/bin/ssh -N -L 15432:db.internal:5432 user@bastion
Restart=always
```

Funciona por uma semana. Aí acontece um desses:

1. A porta remota `5432` cai pra manutenção. A control connection do SSH continua de pé. O forward local tá morto mas o `Restart=always` nunca dispara porque o processo ssh não saiu. **A unit fica "active" mas nada flui.**
2. O roteador do peer reinicia. Keepalive SSH é desligado por default no client OpenSSH, então a sessão TCP fica em `ESTABLISHED` por minutos (às vezes horas, dependendo do `tcp_keepalive_time`) até o kernel finalmente expirar.
3. Tem WireGuard entre você e o bastion. No boot, a unit do systemd corre contra o `wg-quick` e tenta ssh por uma interface que ainda não existe — ssh sai, systemd reinicia, ssh sai de novo, em loop apertado por 30 segundos até o backoff acionar.

A maioria dos guias "SSH tunnel as a service" na internet tem pelo menos um desses bugs.

```bash
matheus@devops:~$ cat o-fix.txt
```

Duas flags SSH e uma dependência do systemd:

| Setting | O que faz | Sem isso |
|---|---|---|
| `-o ExitOnForwardFailure=yes` | Se o forward falha em bindar no start (ou em qualquer renegociação), ssh sai não-zero | Túnel "up" mas morto |
| `-o ServerAliveInterval=30` + `ServerAliveCountMax=3` | Manda probe de keepalive a cada 30s, mata a sessão depois de 3 perdidos (= 90s) | Trava por `tcp_keepalive_time` (2h default no Linux) |
| `After=wg-quick@wgX.service` + `Requires=wg-quick@wgX.service` | systemd espera a interface WireGuard antes de lançar o ssh | Loop de boot até o backoff |
| `-o ConnectTimeout=15` | Se o handshake TCP inicial demora > 15s, desiste | Trava em hairpin / peer fora |
| `Restart=always` + `RestartSec=10` | Se ssh sair (por qualquer motivo), reinicia em 10s | Falha one-shot fica falhada |

Resultado: o túnel **ou tá fluindo, ou o systemd tá ativamente tentando ressuscitar**. Não tem terceiro estado.

```bash
matheus@devops:~$ ls examples/
```

### `examples/simple-tunnel.service`

Túnel SSH puro daqui pra um banco remoto via bastion. Sem VPN no meio. Usa quando:

- O bastion é alcançável na internet pública (ou numa VPC que você já confia)
- Você só precisa de um proxy estável `127.0.0.1:LOCAL_PORT` pra algo dentro da rede remota

```
[this host] ──ssh──► [bastion] ──tcp──► [10.0.0.10:5432]
   ▲
   app local nesse host bate em 127.0.0.1:15432
```

### `examples/wg-then-tunnel.service`

Mesmo túnel SSH, mas o bastion só é alcançável **via WireGuard**. Caso típico de:

- O entry point "público" do peer é ele mesmo um endpoint WireGuard (típico de VPN client-site)
- A LAN remota que você precisa (`192.168.1.0/24` no exemplo) só é alcançável pelo peer WireGuard

```
[this host] ──wg0──► [10.10.0.2 jumpbox] ──ssh──► [192.168.1.10:5432]
   ▲
   esse host bate em 127.0.0.1:15432
```

As linhas críticas no topo da unit:

```ini
After=network-online.target wg-quick@wg0.service
Requires=wg-quick@wg0.service
```

`Requires` faz o systemd se recusar a iniciar o túnel se `wg-quick@wg0` falhou no start. `After` faz o systemd esperar o `wg-quick@wg0` **terminar** a ativação antes de lançar o ssh. Precisa dos dois.

```bash
matheus@devops:~$ ./install.sh
```

```bash
# 1. Copia o exemplo que casa com seu caso
sudo cp examples/simple-tunnel.service /etc/systemd/system/my-tunnel.service

# 2. Edita as linhas Environment= (user, host, portas, caminho da chave)
sudo systemctl edit --full my-tunnel.service

# 3. Garante que a chave ssh existe e tá com o User= que você passou
sudo -u tunneluser ssh-keygen -t ed25519 -f /home/tunneluser/.ssh/tunnel_key -N ''

# 4. Primeira conexão: aceita o host key uma vez (pra StrictHostKeyChecking=accept-new poder escrever em known_hosts)
sudo -u tunneluser ssh -i /home/tunneluser/.ssh/tunnel_key -p 22 bastion@bastion.example.com true

# 5. Start e enable
sudo systemctl daemon-reload
sudo systemctl enable --now my-tunnel.service
sudo systemctl status my-tunnel.service
```

```bash
matheus@devops:~$ cat verificando.txt
```

`systemctl status` só te diz que o processo ssh tá rodando. Pra saber se o forward tá realmente fluindo:

```bash
# A porta local tá em listen?
ss -tlnp | grep 15432

# Você consegue falar com ela?
nc -zv 127.0.0.1 15432

# Quantos forwards a sessão ssh tá de fato proxiando?
sudo ss -tnp | grep " ssh.*pid=\$(pgrep -f 'ssh -N.*-L 15432') "
```

Pra banco, o teste de verdade é um connect one-shot:

```bash
PGPASSWORD=... psql -h 127.0.0.1 -p 15432 -U appuser -d mydb -c 'select 1'
```

```bash
matheus@devops:~$ cat hardening.txt
```

O exemplo usa `StrictHostKeyChecking=accept-new` — isso escreve a chave do host em `~/.ssh/known_hosts` só na **primeira** conexão, e depois enforça estritamente. Mais seguro que `no` (TOFU em vez de "nunca verifica"). Se você consegue pre-provisionar `known_hosts` (ex.: via config management), usa `StrictHostKeyChecking=yes` e pula o passo da primeira conexão no install.

A chave SSH deve ser uma **chave dedicada só pra esse túnel**, não a chave geral do usuário. No lado do bastion, restringe no `authorized_keys`:

```
no-pty,no-X11-forwarding,no-agent-forwarding,permitopen="10.0.0.10:5432" ssh-ed25519 AAAA... tunnel-key
```

`permitopen` faz a chave ser inútil pra qualquer coisa que não seja forward pra esse endereço — então mesmo que a chave vaze, o blast radius é uma porta numa máquina.

```bash
matheus@devops:~$ cat LICENSE
```

MIT. Veja [LICENSE](LICENSE).

```bash
matheus@devops:~$ contact
```

[![LinkedIn](https://img.shields.io/badge/-LinkedIn-0d1117?style=for-the-badge&logo=linkedin&logoColor=39d353)](https://www.linkedin.com/in/matheus-henrique-prates-586328234/)
[![Email](https://img.shields.io/badge/-Email-0d1117?style=for-the-badge&logo=gmail&logoColor=39d353)](mailto:mathues12398henrique@gmail.com)

```bash
matheus@devops:~$ _
```

---

<a id="-english"></a>

## EN

```bash
matheus@devops:~$ cat about.txt
```

A systemd unit template for **SSH tunnels that actually survive failures** — without ending up in the "ssh process exists but the forward is dead" zombie state that most tutorials produce.

```bash
matheus@devops:~$ ls stack/
```

![SSH](https://img.shields.io/badge/-SSH-0d1117?style=for-the-badge&logo=openssh&logoColor=39d353) ![systemd](https://img.shields.io/badge/-systemd-0d1117?style=for-the-badge&logo=systemd&logoColor=39d353) ![Linux](https://img.shields.io/badge/-Linux-0d1117?style=for-the-badge&logo=linux&logoColor=39d353) ![WireGuard](https://img.shields.io/badge/-WireGuard-0d1117?style=for-the-badge&logo=wireguard&logoColor=39d353) ![Bash](https://img.shields.io/badge/-Bash-0d1117?style=for-the-badge&logo=gnubash&logoColor=39d353)

```bash
matheus@devops:~$ cat why-this-exists.txt
```

A naive systemd unit for an SSH tunnel looks like `ExecStart=/usr/bin/ssh -N -L 15432:db.internal:5432 user@bastion` with `Restart=always`. It works for about a week. Then one of these happens:

1. The remote port `5432` goes down for maintenance. The SSH control connection stays up. The local forward is dead but `Restart=always` never fires because the ssh process didn't exit. **The unit is "active" but nothing flows.**
2. The peer router reboots. SSH keepalives are off by default in OpenSSH client, so the TCP session sits in `ESTABLISHED` for minutes (sometimes hours, depending on `tcp_keepalive_time`) before the kernel finally times it out.
3. WireGuard is between you and the bastion. On reboot, the systemd unit races `wg-quick` and tries to ssh through an interface that doesn't exist yet.

Most "SSH tunnel as a service" guides on the internet have at least one of these bugs.

```bash
matheus@devops:~$ cat the-fix.txt
```

Two SSH flags and one systemd dependency:

| Setting | What it does | Without it |
|---|---|---|
| `-o ExitOnForwardFailure=yes` | If the forward fails to bind at startup (or any time after, on a re-negotiation), ssh exits non-zero | Tunnel is "up" but dead |
| `-o ServerAliveInterval=30` + `ServerAliveCountMax=3` | Sends a keepalive probe every 30s, kills the session after 3 missed (= 90s) | Hangs for `tcp_keepalive_time` (2h default on Linux) |
| `After=wg-quick@wgX.service` + `Requires=wg-quick@wgX.service` | systemd waits for the WireGuard interface before launching ssh | Boot-time loop until backoff |
| `-o ConnectTimeout=15` | If the initial TCP handshake takes > 15s, give up | Hangs on hairpin / down peers |
| `Restart=always` + `RestartSec=10` | Once ssh exits (for any reason), restart in 10s | One-shot failure stays failed |

Result: the tunnel **either flows or systemd is actively trying to bring it back**. There is no third state.

```bash
matheus@devops:~$ ls examples/
```

### `examples/simple-tunnel.service`

Plain SSH tunnel from this host to a remote database via a bastion. No VPN in the middle.

```
[this host] ──ssh──► [bastion] ──tcp──► [10.0.0.10:5432]
   ▲
   local app on this host hits 127.0.0.1:15432
```

### `examples/wg-then-tunnel.service`

Same SSH tunnel, but the bastion is reachable only **through WireGuard**.

```
[this host] ──wg0──► [10.10.0.2 jumpbox] ──ssh──► [192.168.1.10:5432]
   ▲
   this host hits 127.0.0.1:15432
```

The critical lines are at the top of the unit:

```ini
After=network-online.target wg-quick@wg0.service
Requires=wg-quick@wg0.service
```

```bash
matheus@devops:~$ ./install.sh
```

```bash
# 1. Copy the example that matches your case
sudo cp examples/simple-tunnel.service /etc/systemd/system/my-tunnel.service

# 2. Edit the Environment= lines (user, host, ports, key path)
sudo systemctl edit --full my-tunnel.service

# 3. Make sure the ssh key exists and is owned by the User= you specified
sudo -u tunneluser ssh-keygen -t ed25519 -f /home/tunneluser/.ssh/tunnel_key -N ''

# 4. First connection: accept the host key once
sudo -u tunneluser ssh -i /home/tunneluser/.ssh/tunnel_key -p 22 bastion@bastion.example.com true

# 5. Start and enable
sudo systemctl daemon-reload
sudo systemctl enable --now my-tunnel.service
sudo systemctl status my-tunnel.service
```

```bash
matheus@devops:~$ cat verifying.txt
```

`systemctl status` only tells you the ssh process is running. To know the forward is flowing:

```bash
ss -tlnp | grep 15432
nc -zv 127.0.0.1 15432
sudo ss -tnp | grep " ssh.*pid=\$(pgrep -f 'ssh -N.*-L 15432') "
PGPASSWORD=... psql -h 127.0.0.1 -p 15432 -U appuser -d mydb -c 'select 1'
```

```bash
matheus@devops:~$ cat hardening.txt
```

The example uses `StrictHostKeyChecking=accept-new` — TOFU on first connect, then strict. Safer than `no`.

Use a **dedicated SSH key for this tunnel only**, restricted in `authorized_keys`:

```
no-pty,no-X11-forwarding,no-agent-forwarding,permitopen="10.0.0.10:5432" ssh-ed25519 AAAA... tunnel-key
```

`permitopen` makes the key useless for anything but forwarding to that one address.

```bash
matheus@devops:~$ cat LICENSE
```

MIT. See [LICENSE](LICENSE).

```bash
matheus@devops:~$ contact
```

[![LinkedIn](https://img.shields.io/badge/-LinkedIn-0d1117?style=for-the-badge&logo=linkedin&logoColor=39d353)](https://www.linkedin.com/in/matheus-henrique-prates-586328234/) [![Email](https://img.shields.io/badge/-Email-0d1117?style=for-the-badge&logo=gmail&logoColor=39d353)](mailto:mathues12398henrique@gmail.com)

```bash
matheus@devops:~$ _
```

<p align="center">
  <img src="https://capsule-render.vercel.app/api?type=waving&color=0:39d353,100:0d1117&height=120&section=footer" width="100%" />
</p>

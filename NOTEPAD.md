# Born2beRoot – Comandos para a avaliação
## 0. Preliminares (na máquina host, VM desligada)
- Calcular a assinatura do disco e comparar:
  ```bash
  cd ~/goinfre                          # vai para a pasta onde está o disco da VM
  sha1sum <VM>.vdi                      # na minha máquina: calcula o hash SHA1 do disco
  cat signature.txt                     # na máquina do avaliador: mostra o hash entregue no repo
  ```
- Confirmar que a VM não está a correr e que não há snapshots:
  ```bash
  VBoxManage list runningvms ⚠️ A VERIFICAR ⚠️       # lista as VMs ligadas (deve vir vazio)
  VBoxManage snapshot "<VM>" list ⚠️ A VERIFICAR ⚠️  # lista os snapshots da VM (não deve haver nenhum)
  ```
- Clonar VM dentro do Oracle VirtualBox: ⚠️ A VERIFICAR ⚠️
  <!-- pista: botão direito na VM → Clonar; arrancar só o clone para o .vdi original não mudar -->
## 2. Simple setup (dentro da VM)
- SO, UFW e SSH:
  ```bash
  head -n 2 /etc/os-release             # mostra a versão do sistema instalado
  sudo systemctl status ufw             # mostra se o serviço da firewall está ativo
  sudo ufw status                       # mostra se a firewall está ativa e as regras
  sudo systemctl status ssh             # mostra se o serviço SSH está ativo
  sudo aa-status                        # mostra se o AppArmor está carregado
  ```
## 3. User step 1 – utilizador e política de passwords
- User do avaliado e grupos:
  ```bash
  id <login>                            # mostra o UID, GID e grupos do user
  groups <login>                        # lista os grupos do user (deve ter sudo e user42)
  getent group sudo user42              # mostra quem pertence aos grupos sudo e user42
  ```
- Criar novo user:
  ```bash
  sudo adduser <novo_user>              # cria o user, a home e pede a password (Rocky: useradd + passwd)
  sudo chage -l <novo_user>             # mostra as regras de expiração da password (máx 30, mín 2, aviso 7)
  ```
- Ver as regras configuradas:
  ```bash
  grep "^PASS_" /etc/login.defs         # mostra os dias de expiração definidos para novos users
  cat /etc/pam.d/common-password        # mostra as regras de força da password (pam_pwquality)
  sudo chage -l <login>                 # confirma a expiração no meu user
  sudo chage -l root                    # confirma a expiração no root
  ```
- Testar a política com uma password fraca:
  ```bash
  su - <novo_user>                      # entra como o novo user
  passwd                                # tenta mudar a password (testar "abc", "aaaa1234Ab", etc.)
  exit                                  # volta ao user anterior
  ```
## 4. User step 2 – grupo "evaluating"
```bash
sudo groupadd evaluating                # cria o grupo evaluating
sudo usermod -aG evaluating <novo_user> # adiciona o user ao grupo sem o tirar dos outros
getent group evaluating                 # mostra os membros do grupo
groups <novo_user>                      # confirma os grupos do user
```
## 5. Hostname e partições
- Ver e alterar hostname:
  ```bash
  hostnamectl                           # mostra o hostname atual e info do sistema
  sudo hostnamectl set-hostname <teu_login>42  # muda o hostname
  sudo nano /etc/hosts                  # troca o nome antigo na linha 127.0.1.1
  sudo reboot                           # reinicia a VM
  hostname                              # confirma o novo nome depois do reboot
  ```
- Repor o original (mesmos passos com `<login>42` + reboot).
- Partições e LVM:
  ```bash
  lsblk                                 # mostra discos, partições, encriptação e volumes LVM
  sudo pvs && sudo vgs && sudo lvs      # mostra volumes físicos, grupos e volumes lógicos do LVM
  ```
## 6. SUDO
- Instalação e novo user no grupo sudo:
  ```bash
  dpkg -l | grep sudo                   # confirma que o pacote sudo está instalado
  sudo --version                        # mostra a versão do sudo
  sudo usermod -aG sudo <novo_user>     # adiciona o novo user ao grupo sudo
  getent group sudo                     # confirma os membros do grupo sudo
  ```
- Mostrar as regras (passwd_tries=3, badpass_message, logfile, log_input, log_output, iolog_dir, requiretty, secure_path):
  ```bash
  sudo cat /etc/sudoers.d/*             # mostra as regras do sudo (ou: sudo visudo para editar com segurança)
  ```
- Logs:
  ```bash
  sudo ls -la /var/log/sudo/            # lista os ficheiros de log do sudo
  sudo cat /var/log/sudo/sudo.log       # mostra o histórico de comandos feitos com sudo
  sudo ls /var/log/sudo/00/00/          # lista as gravações de input/output
  ```
- Testar e confirmar que o log atualiza:
  ```bash
  sudo -k && sudo ls                    # esquece a password guardada; errar 3x para ver a mensagem e o limite
  sudo ls /root                         # corre um comando com sudo para gerar um novo log
  sudo tail -n 5 /var/log/sudo/sudo.log # mostra as últimas 5 linhas do log (deve aparecer o comando)
  ```
## 7. UFW / Firewalld
```bash
dpkg -l | grep ufw                      # confirma que o UFW está instalado
sudo ufw status verbose                 # mostra o estado e as regras com detalhe
sudo ufw status numbered                # mostra as regras numeradas (deve haver 4242)
sudo ufw allow 8080                     # abre a porta 8080
sudo ufw status numbered                # confirma que a regra 8080 foi adicionada
sudo ufw delete allow 8080              # apaga a regra da porta 8080
sudo ufw status numbered                # confirma que a regra foi removida
```
> Rocky: `sudo firewall-cmd --list-all` · `sudo firewall-cmd --add-port=8080/tcp` · `sudo firewall-cmd --remove-port=8080/tcp`
## 8. SSH
- Na VM:
  ```bash
  dpkg -l | grep openssh-server         # confirma que o servidor SSH está instalado
  sudo systemctl status ssh             # mostra se o SSH está ativo
  sudo grep -E "^Port|^PermitRootLogin" /etc/ssh/sshd_config  # mostra a porta (4242) e se o root está bloqueado
  sudo ss -tunlp | grep ssh             # mostra em que portas o SSH está a escutar (só 4242)
  hostname -I                           # mostra o IP da VM
  ```
- No host (com port forwarding 4242 no VirtualBox):
  ```bash
  ssh <novo_user>@localhost -p 4242     # liga por SSH com o novo user (deve funcionar)
  exit                                  # sai da sessão SSH
  ssh root@localhost -p 4242            # tenta ligar como root (deve ser recusado)
  ```
## 9. Script monitoring.sh
- Ver código, permissões e assinatura (para provar depois que não mudou):
  ```bash
  sudo crontab -u root -l               # mostra as tarefas agendadas do root (onde está o script)
  ls -l /usr/local/bin/monitoring.sh    # mostra as permissões do script
  cat /usr/local/bin/monitoring.sh      # mostra o código do script
  sha1sum /usr/local/bin/monitoring.sh  # tira o hash do script para comparar depois
  ```
- Passar a correr a cada minuto:
  ```bash
  sudo crontab -u root -e               # edita o crontab: trocar */10 * * * * por * * * * *
  ```
- Testar valores dinâmicos:
  ```bash
  yes > /dev/null &                     # põe um processo em fundo a gastar CPU
  kill %1                               # mata esse processo
  sudo ls                               # incrementa o contador de comandos sudo
  ```
- Parar o script sem o modificar:
  ```bash
  sudo systemctl disable cron && sudo systemctl stop cron  # desliga o cron agora e no arranque
  # alternativa: comentar a linha no crontab
  sudo reboot                           # reinicia para provar que o script já não corre
  ```
- Depois do reboot, confirmar que nada mudou:
  ```bash
  ls -l /usr/local/bin/monitoring.sh    # confirma que o script existe e as permissões são iguais
  sha1sum /usr/local/bin/monitoring.sh  # confirma que o hash é igual ao de antes
  ```
- Repor: `sudo systemctl enable --now cron` (volta a ligar o cron) e voltar a `*/10` no crontab.
## 10. Bónus (só se o mandatório estiver perfeito)
```bash
lsblk                                   # mostra as partições do bónus
sudo systemctl status lighttpd mariadb  # mostra se o servidor web e a base de dados estão ativos
php -v                                  # mostra a versão do PHP instalada
dpkg -l | grep -E "nginx|apache2"       # confirma que NGINX/Apache2 não existem (deve vir vazio)
sudo ufw status                         # mostra as portas extra abertas (ex: 80)
sudo mariadb -e "SHOW DATABASES;"       # lista as bases de dados (deve haver a do WordPress)
sudo systemctl status <servico_extra>   # mostra se o serviço extra está ativo
```
- WordPress no browser do host: `http://localhost:<porta_forward>`
## 11. Fim (no host, VM desligada)
```bash
VBoxManage snapshot "<VM>" delete avaliacao  # apaga o snapshot da avaliação
VBoxManage snapshot "<VM>" list              # confirma que não sobrou nenhum snapshot
sha1sum <VM>.vdi                             # confirma que a assinatura se mantém
```
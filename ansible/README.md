# Ubuntu Kiosk com Chrome e Openbox

Referência Ansible para configurar um Ubuntu Server como kiosk de uma página do
Uptime Kuma.

## O que é configurado

- Xorg, Openbox, `unclutter` e utilitários X11.
- Google Chrome estável via pacote `.deb`, sem Snap.
- Login automático no `tty1`.
- Inicialização automática do Xorg/Openbox.
- Chrome em tela cheia, modo escuro, sem notificações e com reinício automático.
- Desativação de suspensão, blanking e DPMS.
- Desativação dos timers de atualização automática.
- Netplan parametrizado, opcional, para IP estático, gateway e DNS.

## Uso

Edite `group_vars/kiosk.yml`:

```yaml
kiosk_user: kiosk
kuma_url: "http://IP-DO-KUMA:3001/status/SEU-SLUG"
```

Preencha também `kiosk_network` somente se quiser que o playbook gerencie a
configuração de rede. Por segurança, `kiosk_manage_network` é `false` por
padrão.

Execute a partir deste diretório:

```bash
ansible-playbook -i inventory.ini site.yml -K
```

O pacote do Chrome é baixado durante a execução de
`google-chrome-stable_current_amd64.deb`. A máquina precisa ter acesso à
Internet ou o arquivo deve estar disponível em `chrome_deb_src`.

Após a execução, reinicie a máquina para validar o fluxo completo de autologin:

```bash
sudo reboot
```

Altere a senha do usuário kiosk antes de colocar o equipamento em produção.

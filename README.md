# Ubuntu Kiosk com Chrome e Openbox

Playbook Ansible para transformar um Ubuntu Server em um terminal dedicado à
exibição de uma Status Page do Uptime Kuma. O objetivo é ter uma máquina que,
após ligar, mostre somente a página monitorada e recupere o navegador se ele
for encerrado.

## Decisões de projeto

### Ubuntu Server como base

O projeto parte de uma instalação Server para reduzir o número de pacotes,
serviços e superfícies de configuração. A interface gráfica necessária para o
kiosk é instalada explicitamente pelo playbook (`xorg` e `openbox`), em vez de
depender de um desktop completo.

O playbook valida `ansible_distribution == "Ubuntu"` antes de alterar a
máquina. Isso evita aplicar nomes de serviços e convenções de pacotes em uma
distribuição não testada.

### Xorg + Openbox, sem desktop completo

`startx` é iniciado somente no `tty1` pelo `.bash_profile` do usuário
dedicado. O Openbox fornece a sessão gráfica mínima, sem painel, gerenciador
de arquivos ou aplicativos que possam tirar o foco da página.

Essa composição foi escolhida por ser simples de diagnosticar: o console
continua disponível para manutenção e não há um display manager adicional
introduzindo outra etapa de login ou configuração.

### Autologin no `tty1`

O override de `getty@tty1.service` faz login automático como `kiosk`. Em
seguida, o perfil desse usuário executa `startx`. Assim o boot não depende de
interação humana, mas o acesso físico ao console continua sendo um risco:
autologin não é uma medida de segurança.

### Chrome instalado por `.deb`, sem Snap

O Chrome estável é baixado da URL oficial e instalado com o módulo `apt` a
partir do pacote `.deb`. A escolha evita adicionar a camada de confinamento e
o tempo de inicialização de um Snap, mantendo o caminho do executável
(`/usr/bin/google-chrome`) explícito no serviço.

O arquivo temporário é removido após a instalação para não deixar artefatos
desnecessários no disco. A URL aponta para `amd64`; máquinas de outra
arquitetura precisam de uma fonte e pacote compatíveis antes de usar este
playbook.

### Serviço systemd no escopo do usuário

O navegador é um `kiosk-browser.service` do usuário, e não um processo
iniciado diretamente pelo Openbox. Isso fornece reinício automático
(`Restart=always`), uma unidade observável pelo `systemctl --user` e uma
separação clara entre a sessão gráfica e o aplicativo.

`loginctl enable-linger` mantém o gerenciador systemd do usuário disponível
entre logins. O serviço ainda depende de `DISPLAY=:0` e do `.Xauthority`
criados pela sessão X; por isso o autostart do Openbox solicita um restart
quando a sessão gráfica está pronta.

As opções do Chrome desativam diálogos, notificações, gestos e avisos de
sessão para evitar que a interface de operação fique visível ao público.
`--kiosk` remove a moldura e os controles do navegador. O modo escuro é uma
preferência visual para telas de monitoramento, não um requisito do Uptime
Kuma.

### Desativação de blanking e DPMS

O autostart executa `xset` para impedir protetor de tela, blanking e desligamento
do monitor. `unclutter` esconde o cursor depois de um segundo de inatividade.
Essas escolhas são intencionais para um painel sempre visível; em hardware
alimentado por bateria ou sujeito a retenção de imagem, elas devem ser
reavaliadas.

### Atualizações automáticas desativadas

Os timers `apt-daily`, `apt-daily-upgrade` e o serviço de unattended upgrades
são parados para evitar reinicializações, troca de pacotes ou perda de
conectividade durante a exibição. Isso privilegia previsibilidade do painel,
mas aumenta a responsabilidade operacional: atualizações de segurança devem
ser planejadas e executadas manualmente, com uma janela de manutenção e
validação após o reboot.

### Rede opt-in

Por padrão, `kiosk_manage_network: false` preserva a rede já configurada no
equipamento. O playbook só grava `/etc/netplan/99-kiosk.yaml` e executa
`netplan apply` quando essa opção é habilitada. Essa proteção evita derrubar a
conexão SSH ou sobrescrever uma configuração gerenciada por outra ferramenta.

Quando habilitada, a rede é declarada de forma determinística com IP estático,
rota padrão e DNS. Os valores de exemplo usam a faixa reservada
`192.0.2.0/24` e não funcionam como endereços reais.

## Estrutura

| Caminho | Responsabilidade |
| --- | --- |
| `site.yml` | Valida o alvo e aplica pacotes, sessão gráfica, serviços e rede |
| `group_vars/kiosk.yml` | Variáveis e valores editáveis da implantação |
| `inventory.ini` | Hosts e credenciais SSH usados pelo Ansible |
| `templates/getty-autologin.conf.j2` | Autologin no `tty1` |
| `templates/bash_profile.j2` | Início do Xorg apenas no `tty1` |
| `templates/openbox-autostart.j2` | Ajustes da tela e início do serviço do navegador |
| `templates/kiosk-browser.service.j2` | Execução e recuperação do Chrome |
| `templates/netplan-kiosk.yaml.j2` | Configuração opcional de rede estática |

## Configuração e execução

Edite `inventory.ini` com o endereço e o usuário SSH do equipamento. Depois,
ajuste pelo menos:

```yaml
kiosk_user: kiosk
kuma_url: "http://IP-DO-KUMA:3001/status/SEU-SLUG"
```

`kuma_url` precisa começar com `http://` ou `https://`; o playbook interrompe
a execução para valores inválidos. O usuário informado deve existir, ter como
home o valor esperado em `kiosk_home` e ser capaz de iniciar uma sessão local.

Para deixar o playbook gerenciar a rede, altere a opção e preencha todos os
valores de `kiosk_network`:

```yaml
kiosk_manage_network: true
kiosk_network:
  interface: eno1
  address: 192.0.2.10/24
  gateway: 192.0.2.1
  dns:
    - 192.0.2.1
    - 1.1.1.1
  search: []
```

Execute a partir deste diretório:

```bash
ansible-playbook -i inventory.ini site.yml -K
```

O alvo precisa ter acesso à Internet para baixar o Chrome da URL definida em
`chrome_deb_url`. O playbook não implementa um fluxo offline; nesse caso, é
necessário disponibilizar um repositório ou adaptar a tarefa de instalação.

Reinicie para validar o fluxo completo de boot, autologin e sessão gráfica:

```bash
sudo reboot
```

## Operação e segurança

Antes de colocar o equipamento em produção:

- substitua os valores de exemplo do inventário, URL, rede e senha;
- use HTTPS para o Uptime Kuma quando a página estiver disponível dessa forma;
- limite o acesso SSH e físico ao equipamento;
- defina uma rotina de atualização manual, já que os timers automáticos foram
  desativados;
- verifique o estado com `systemctl --user status kiosk-browser.service` como
  o usuário `kiosk` e consulte os logs com
  `journalctl --user -u kiosk-browser.service`.

O kiosk não é um mecanismo de isolamento forte: quem obtiver acesso root ou
ao console físico poderá alterar a configuração ou sair da aplicação.

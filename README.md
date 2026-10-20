# ephemeral-vm-lab

Disposable QEMU/KVM VMs with a fresh identity per session, per-session nftables isolation and a RAM-only disk. Bash + libvirt + cloud-init.

> Laboratório de testes em que **cada sessão parte de uma imagem limpa**, com identidade de rede nova, isolada do host e da LAN. Ao sair, tudo é destruído. Projeto de estudo (Blue Team), não é uma ferramenta de anonimato: veja [Limitações](#limitações).

## Como funciona

| Componente | Detalhe |
|---|---|
| Conexão libvirt | `qemu:///system` (QEMU com AppArmor/sVirt, não roda como usuário comum) |
| Base | Imagem qcow2 somente leitura (`golden`), criada a partir de uma instalação limpa do Ubuntu Desktop |
| Disco da sessão | Overlay qcow2 em **tmpfs** (RAM) sobre a base. Nada é gravado em disco |
| Rede | Rede transitória por sessão, sub-rede `10.X.Y.0/24` sorteada e conferida contra as rotas do host |
| Firewall | Tabela **nftables** própria por sessão: a VM alcança só DNS/DHCP do host e a internet pública |
| Identidade nova | hostname, MAC, IP, instance-id, host keys SSH, machine-id, fuso e idioma |
| Superfície reduzida | Sem USB, som e balloon; SPICE por socket Unix (sem porta TCP); clipboard e transferência de arquivos desligados |
| Autodestruição | `--ttl` (padrão 8h) e `--cleanup-all` para restos de queda de energia ou `kill -9` |

## Pré-requisitos

- Ubuntu/Debian com aceleração KVM (`/dev/kvm`)
- Usuário com `sudo` (o script cria rede, regras de firewall e monta tmpfs)
- RAM livre ≥ `--ram` + `--scratch` (o disco da sessão ocupa memória)
- Recomendado: `sudo swapoff -a` antes de sessões sensíveis, para a RAM não vazar para o swap
- `virt-viewer` para abrir a tela

## 1. Criar a base limpa

1. No virt-manager, crie uma VM a partir do ISO do Ubuntu Desktop, com firmware **BIOS**.
2. Instalação mínima, usuário `templab` (o autologin usa esse nome), senha simples.
3. **Não abra o navegador** nem entre em contas: tudo que ficar na base aparece em todas as sessões.
4. Opcional, para o login automático: confirme em `/etc/gdm3/custom.conf`:
   ```ini
   [daemon]
   AutomaticLoginEnable=true
   AutomaticLogin=templab
   ```
5. Prepare e desligue:
   ```bash
   sudo apt update && sudo apt upgrade -y
   sudo apt install -y cloud-init openssh-server
   sudo cloud-init clean --logs --seed
   sudo truncate -s 0 /etc/machine-id
   sudo shutdown now
   ```
6. Guarde uma cópia do disco como backup.

## 2. Importar a base

```bash
sudo ./ephemeral-vm.sh --import-golden /caminho/para/BaseLimpa.qcow2
```

O disco é convertido para uma imagem somente leitura em `/var/lib/libvirt/images/ephemeral/`.

## 3. Rodar uma sessão

```bash
# Desktop com tela
sudo ./ephemeral-vm.sh --golden BaseLimpa --gui --ram 4096 --scratch 4

# Atalho (opcional)
echo "alias vmtemp='sudo ~/ephemeral-vm.sh --golden BaseLimpa --gui --ram 4096 --scratch 4'" >> ~/.bashrc
```

Quando o banner **VM PRONTA** aparecer, abra a tela em outro terminal:

```bash
virt-viewer --attach -c qemu:///system eph-XXXXXXXX
```

O `--attach` é obrigatório porque o SPICE usa socket Unix. Para encerrar: `Ctrl+C` no terminal do script.

Outras opções:

```bash
sudo ./ephemeral-vm.sh                 # shell SSH numa VM servidor (imagem cloud do Ubuntu)
sudo ./ephemeral-vm.sh --no-net        # sem internet
sudo ./ephemeral-vm.sh --cleanup-all   # remove VMs/redes/regras/tmpfs órfãos
```

## 4. Validar o isolamento

Dentro da VM (troque pelos IPs reais):

```bash
curl -sI https://ubuntu.com | head -1                          # deve responder 200
ping -c1 -W2 IP_DO_ROTEADOR || echo "bloqueado"                # deve bloquear
timeout 3 bash -c '</dev/tcp/GATEWAY_DA_VM/22' && echo "ACESSOU (ruim)" || echo "bloqueado (bom)"
```

Faça antes um controle no host: o ping ao roteador precisa funcionar, senão o teste dentro da VM não prova nada.

## 5. Confirmar a limpeza

No host, depois de encerrar:

```bash
sudo virsh -c qemu:///system list --all
sudo virsh -c qemu:///system net-list --all
sudo nft list tables | grep eph_ || echo "limpo"
mount | grep ephemeral || echo "tmpfs desmontado"
```

## Problemas encontrados e soluções

| Problema | Causa | Solução |
|---|---|---|
| `sudo` recusado | Perfil de estudos sem privilégio de root | Rodar tudo pelo perfil administrador |
| Disco da base não existia | O XML apontava para um disco de sessão já apagado | Descobrir o disco real com `virsh domblklist` e importar uma **cópia** |
| `virt-viewer` sem janela | A sessão gráfica pertence a outro usuário | Entrar pela tela de login, ou `xhost +SI:localuser:USUARIO` com `DISPLAY=:0` |
| "A exibição pode ser anexada somente através do libvirt" | SPICE sem porta TCP | Usar `virt-viewer --attach` |
| Autologin intermitente | O GDM sobe antes de o cloud-init terminar e ignora a config | O script espera o SSH responder e então roda `systemctl restart gdm` |
| `=#!/usr/bin/env: not found` | Caractere sobrando antes do shebang, e o arquivo rodou com `sh` | Corrigir a linha 1. O `bash -n` não detecta esse caso |
| Tmpfs e VM que sobram após queda | Sessão interrompida sem passar pelo `trap` | `--cleanup-all` |

## O que muda a cada sessão e o que não muda

| Muda | Vem da base (igual em toda sessão) |
|---|---|
| Hostname, MAC, IP e sub-rede | Versão do Firefox e dos pacotes |
| machine-id e chaves SSH | Fontes, resolução, renderizador de vídeo |
| Fuso horário e idioma (sorteados) | Usuário e senha do desktop |
| Cookies, histórico, arquivos da sessão | Hardware virtual |

## Limitações

- **Não esconde o IP público**: a VM sai pela mesma conexão do host. Para isso é preciso VPN ou Tor.
- **Não evita fingerprint do navegador.** Como todas as sessões saem da mesma imagem, os sinais (versão, fontes, WebGL, hardware virtual) são praticamente idênticos entre elas. Para reduzir, o caminho é padronizar o fingerprint (Mullvad Browser ou Tor Browser), não variar o estado.
- O disco da sessão ocupa RAM. Uma VM desktop com 4 GB de RAM e 4 GB de scratch pode usar até 8 GB do host.
- Uma falha de escape do QEMU/KVM continua possível: mantenha host e QEMU atualizados. Para malware sério, considere Whonix, Tails ou Qubes.
- O script foi validado de forma prática em um único host (Ubuntu), com `bash -n`, e não por uma suíte de testes.

## Manutenção da base

- Não ligue a base fora de uma atualização.
- Para atualizar: ligue a base, atualize, repita `cloud-init clean --logs --seed` e `truncate -s 0 /etc/machine-id`, desligue e rode `--import-golden` de novo.
- Mantenha um backup do `.qcow2` da base e do script.

## Aprendizados

- Isolamento de rede com nftables (cadeias `input` e `forward` por interface da bridge)
- Ciclo de vida de VMs transitórias no libvirt e limpeza idempotente com `trap`
- Cloud-init: identidade por instância, ordem de boot e interação com o GDM
- Diferença entre descartar **estado** e controlar **fingerprint**

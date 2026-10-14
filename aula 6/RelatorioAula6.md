# Relatório Técnico - Aula Prática 06

## 1. Identificação
- Título da prática: Configuração de Rede Estática com Netplan e Modo Placa em Ponte (Bridge Adapter) no VirtualBox
- Aluno: João Pedro de Castro Laranjeira Rocha
- Matrícula: 2023007098
- Turma: LSOR - BSI
- Data da prática: 23/09/2026

## 2. Objetivo
Trocar o adaptador de rede da máquina virtual Ubuntu Server do modo NAT para o modo Placa em Ponte (Bridge), fazendo com que a VM passe a ser um host da própria rede física do laboratório do IFAL (`172.20.20.0/22`). Em seguida, configurar um endereço IP estático, gateway e servidores DNS através do Netplan (`/etc/netplan/00-installer-config.yaml`) e validar a conectividade com testes de ping bidirecional e `traceroute`.

## 3. Ambiente
- Host: computador Windows do laboratório, com IP `172.20.21.218` na rede do IFAL
- Hipervisor: Oracle VirtualBox
- Máquina virtual: `ubuntu_server_basic` (hostname `ubuntu-server`)
- Sistema operacional da VM: Ubuntu Server 26.04 LTS
- Usuário administrativo utilizado: `administrador`
- Adaptador 1 da VM: Placa em modo Bridge ligada à placa física `Realtek PCIe GbE Family Controller` (interface `enp0s3` no Linux)

### Parâmetros de rede utilizados
| Parâmetro | Valor |
| :--- | :--- |
| Rede do laboratório | `172.20.20.0/22` (`255.255.252.0`) |
| IP estático da VM | `172.20.23.20/22` |
| Gateway padrão | `172.20.20.1` |
| Servidores DNS | `172.20.20.1`, `1.1.1.1`, `8.8.8.8` |

## 4. Procedimento
1. No PowerShell do Windows, foi executado `ping 172.20.23.20` para verificar se o endereço candidato estava livre. Como nenhum equipamento respondeu, o IP foi escolhido para a VM.
2. No VirtualBox, em **Configurações -> Rede -> Adaptador 1**, o modo foi alterado de NAT para **Placa em modo Bridge**, selecionando a placa cabeada `Realtek PCIe GbE Family Controller` e mantendo o cabo virtual conectado.
3. Na VM, o arquivo `/etc/netplan/00-installer-config.yaml` foi editado com `sudo nano`. A configuração original gerada pelo instalador (`subiquity`, com DHCP) foi mantida apenas como comentário, e foi adicionada a configuração estática:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 172.20.23.20/22
      routes:
        - to: default
          via: 172.20.20.1
      nameservers:
        addresses:
          - 172.20.20.1
          - 1.1.1.1
          - 8.8.8.8
```

4. O conteúdo foi conferido com `sudo cat /etc/netplan/00-installer-config.yaml`.
5. A configuração foi aplicada com `sudo netplan apply` e verificada com `ip addr show enp0s3`.
6. Foram executados os testes de ping do Windows para a VM e da VM para o Windows.
7. Por fim, foram executados `traceroute google.com`, `traceroute one.one.one.one` e `traceroute 1.1.1.1` para observar a rota de saída pela rede do IFAL.

### Comandos principais utilizados
*   **`ping`:** envia pacotes ICMP Echo Request e mede se o destino responde.
*   **`nano` / `cat`:** edição e exibição do arquivo YAML do Netplan.
*   **`netplan apply`:** valida o YAML e aplica a configuração de rede ao `systemd-networkd`.
*   **`ip addr show`:** exibe os endereços atribuídos a uma interface.
*   **`traceroute`:** mostra cada salto (roteador) entre a VM e o destino.

## 5. Testes e Evidências

**Print 1 - Teste de IP livre: ping do Windows para 172.20.23.20 antes da configuração**

![Print 1 - Teste de IP livre: ping do Windows para 172.20.23.20 antes da configuração](./1.png)

As respostas vêm do próprio host (`172.20.21.218`) com a mensagem "Host de destino inacessível": o Windows não obteve resposta ARP para o endereço, ou seja, nenhum equipamento estava usando o IP. O Windows contabiliza essas mensagens como "recebidos", por isso a estatística mostra 0% de perda, mas não houve resposta do destino.

**Print 2 - Adaptador 1 da VM configurado em Placa em modo Bridge (Realtek PCIe GbE)**

![Print 2 - Adaptador 1 da VM configurado em Placa em modo Bridge (Realtek PCIe GbE)](./2.png)

**Print 3 - Conteúdo do arquivo /etc/netplan/00-installer-config.yaml com IP estático, rota padrão e DNS**

![Print 3 - Conteúdo do arquivo /etc/netplan/00-installer-config.yaml com IP estático, rota padrão e DNS](./3.png)

**Print 4 - Aplicação com netplan apply e confirmação do IP 172.20.23.20/22 na interface enp0s3**

![Print 4 - Aplicação com netplan apply e confirmação do IP 172.20.23.20/22 na interface enp0s3](./4.png)

**Print 5 - Ping do Windows (host) para a VM em 172.20.23.20**

![Print 5 - Ping do Windows (host) para a VM em 172.20.23.20](./5.png)

**Print 6 - Ping da VM para o Windows (host) em 172.20.21.218**

![Print 6 - Ping da VM para o Windows (host) em 172.20.21.218](./6.png)

**Print 7 - traceroute google.com**

![Print 7 - traceroute google.com](./7.png)

**Print 8 - traceroute one.one.one.one e traceroute 1.1.1.1**

![Print 8 - traceroute one.one.one.one e traceroute 1.1.1.1](./8.png)

O ping bidirecional teve 0% de perda, comprovando que a VM passou a fazer parte da rede física do laboratório. O TTL das respostas confirma o sistema de cada lado: 64 na resposta da VM (Linux) e 128 na resposta do host (Windows).

Nos `traceroute`, o primeiro salto é sempre o gateway do laboratório (`_gateway`, `172.20.20.1`), seguido pelos roteadores do provedor (`190.15.107.25` e `10.11.80.1`). Para o Google, a rota termina em um servidor em Guarulhos (`gru14s20-in-f14.1e100.net`). Para a Cloudflare, o nome `one.one.one.one` foi resolvido para `1.1.1.1`, e a rota passa pelo ponto de troca de tráfego de Fortaleza (`as13335.fortaleza.ce.ix.br`). O salto 4 aparece como `* * *` porque esse roteador não responde às sondas do traceroute, o que não impede a chegada ao destino.

## 6. Problemas e Soluções
- Sem problemas para essa etapa. O teste prévio de IP livre evitou conflito de endereço com outras máquinas do laboratório.

## 7. Conclusão
O modo Bridge tornou a VM um host independente da rede do laboratório, acessível diretamente pelo host e pelos demais equipamentos, sem a necessidade de redirecionamento de portas como no modo NAT. A configuração estática com Netplan garante que o servidor tenha sempre o mesmo endereço, o que é essencial para serviços que precisam ser encontrados pela rede. Em contrapartida, IP estático exige controle: é preciso verificar se o endereço está livre antes de usá-lo e manter a máscara, o gateway e o DNS coerentes com a rede onde a máquina está conectada.

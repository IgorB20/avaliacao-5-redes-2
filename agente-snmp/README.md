# Agente SNMP - Ubuntu 22.04 LTS

Este container representa a VM SNMP Agent do estudo dirigido. Ele roda Ubuntu
22.04 com `snmpd` habilitado na porta UDP 161 e comunidade SNMPv2c `public`.

## Subir o ambiente

Primeiro suba o Zabbix a partir da pasta `zabbix-docker`:

```bash
cd ../zabbix-docker
make up
```

Depois suba o agente SNMP:

```bash
cd ../agente-snmp
docker compose up -d --build
```

## Testar pelo host

A porta UDP 161 do container fica publicada no host como UDP 1161:

```bash
snmpwalk -v2c -c public -On udp:127.0.0.1:1161 1.3.6.1.2.1.1
```

O teste do roteiro usando `system` depende das MIBs textuais do cliente
`snmpwalk`. Em Ubuntu/Debian e em imagens Docker minimalistas, essas MIBs
costumam vir desabilitadas ou incompletas. Se aparecer:

```text
system: Unknown Object Identifier (Sub-id not found: (top) -> system)
```

use o OID numerico abaixo, que e equivalente a arvore `system`:

```bash
snmpwalk -v2c -c public udp:127.0.0.1:1161 1.3.6.1.2.1.1
```

Para testar localmente dentro do container:

```bash
docker exec -it snmp-agent snmpwalk -v2c -c public localhost 1.3.6.1.2.1.1
```

Para testar remotamente a partir de outro container na rede do Zabbix:

```bash
docker run --rm --network zabbix-docker_frontend ubuntu:22.04 \
  sh -c "apt-get update >/dev/null && apt-get install -y snmp >/dev/null && snmpwalk -v2c -c public snmp-agent 1.3.6.1.2.1.1"
```

## Cadastrar no Zabbix

No frontend do Zabbix, crie um host com:

- Host name: `SNMP-Agent-1`
- Interface: `SNMP`
- DNS name: `snmp-agent`
- Connect to: `DNS`
- Port: `161`
- SNMP version: `SNMPv2`
- Community: `public`
- Template: `Template Net SNMP Device` ou template SNMP Linux equivalente

Como o container entra na rede Docker `zabbix-docker_frontend`, o Zabbix Server
consegue resolver `snmp-agent` diretamente por DNS interno do Docker.

## Captura Wireshark

No Docker Desktop para macOS, o tráfego encaminhado para containers pode não
aparecer diretamente na interface `lo0` do Wireshark. O caminho mais confiavel
para o trabalho e capturar dentro do proprio container e abrir o arquivo no
Wireshark.

Em um terminal, inicie a captura:

```bash
docker exec -it snmp-agent tcpdump -i any -nn -s0 -w /tmp/snmp.pcap udp port 161
```

Em outro terminal, gere trafego SNMP:

```bash
snmpwalk -v2c -c public udp:127.0.0.1:1161 1.3.6.1.2.1.1
```

Volte ao terminal do `tcpdump` e pare com `Ctrl+C`. Depois copie o arquivo:

```bash
docker cp snmp-agent:/tmp/snmp.pcap ./snmp.pcap
```

Abra `snmp.pcap` no Wireshark e use o filtro de exibicao:

```text
snmp || udp.port == 161
```

Se quiser tentar capturar direto pelo Wireshark no macOS, selecione `Loopback:
lo0`, gere trafego com `snmpwalk` no host e use:

```text
udp.port == 1161
```

As consultas esperadas sao `GET`, `GET-NEXT` e respostas SNMP com OID e valor.

## Alerta com OID SNMP personalizado no Zabbix

Para testar a criacao de alertas com base em OIDs SNMP, foi criado no Zabbix
um item usando o OID `1.3.6.1.2.1.1.3.0`, correspondente ao tempo de atividade
do agente SNMP (`sysUpTime.0`). O item deve ser configurado manualmente, sem
depender apenas dos templates.

No host `SNMP-Agent-1`, crie um item:

- Name: `Custom SNMP uptime`
- Type: `SNMP agent`
- Key: `snmp.custom.uptime`
- SNMP OID: `1.3.6.1.2.1.1.3.0`
- Type of information: `Numeric (unsigned)`
- Update interval: `30s`

Depois crie uma trigger:

- Name: `Sem resposta do OID SNMP customizado`
- Severity: `Warning`
- Expression: `nodata(/SNMP-Agent-1/snmp.custom.uptime,2m)=1`

Para testar o alerta, interrompa o container do agente SNMP:

```bash
docker stop snmp-agent
```

Apos aproximadamente dois minutos sem receber valores desse OID, o Zabbix deve
gerar o alerta. Para normalizar, inicie o container novamente:

```bash
docker start snmp-agent
```

Quando a coleta for restabelecida, o alerta deve retornar ao estado normal.

Texto sugerido para o relatorio:

```text
Para testar a criacao de alertas com base em OIDs SNMP, foi criado no Zabbix um
item utilizando o OID 1.3.6.1.2.1.1.3.0, correspondente ao tempo de atividade
do agente SNMP. O item foi configurado com a chave snmp.custom.uptime e coleta
periodica de dados.

Em seguida, foi criada uma trigger para gerar um alerta caso o Zabbix
permanecesse sem receber valores desse OID por dois minutos:

nodata(/SNMP-Agent-1/snmp.custom.uptime,2m)=1

Para realizar o teste, o container do agente SNMP foi interrompido com o comando
docker stop snmp-agent. Apos o periodo configurado, o Zabbix identificou a
ausencia de respostas e gerou o alerta. Depois que o container foi iniciado
novamente com docker start snmp-agent, a coleta foi restabelecida e o alerta
retornou ao estado normal.
```

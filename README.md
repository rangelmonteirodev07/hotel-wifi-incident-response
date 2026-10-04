# Estudo de Caso: Análise de Riscos, Defesa Ativa e Hardening em Redes e Hospitalidade
**Autor:** Enzo (Raptor)
**Categoria:** Segurança de Redes / Engenharia de Defesa (Blue Team) / Resposta a Incidentes

---

## 1. Sumário Executivo
Este documento reporta uma análise de segurança em redes e resposta a incidentes realizada de forma reativa a partir de um host Linux Local (`Raptor`). Durante uma hospedagem, foram constatadas anomalias severas na infraestrutura de Wi-Fi pública do estabelecimento (vulnerabilidades estruturais, ausência de isolamento e intercepção ativa de tráfego criptografado).

o projeto demonstra a investigação técnica através de análise de pacotes (Wireshark) e varreduras de diagnóstico (Angry IP Scanner), culminando na aplicação prática de técnicas de *Hardening* do sistema operacional para mitigação total de riscos.

---

### 2. Cenário e vetores de Risco Identificados
Ao ingressar na rede sem fio de hóspedes, foram mapeados os seguintes riscos arquiteturais imediatos:
* **Ausência de criptografia em camada de Enlace:** Rede operando em modo aberto (sem WPA2/WPA3), permitindo a interceptação passiva (sniffing) de tráfego por qualquer agente adjacente.
* **Ausência de Isolamento de ponto de acesso (AP Isolation Desativado):** Permissão nativa para comunicação direta *Peer-to-Peer (entre dispositivos dos Hóspedes), Expandindo drasticamente a superfície de ataque interna.

---

## 3. Investigação técnica e coleta e Evidências

### Fase A: Mapeamento de Hosts e Identificação de Defesa Ativa (Angry IP Scanner)
Ao executar varreduras exploratórias na faixa e sub-rede do gateway (`192.168.4.0/22`), o sistema retornou uma resposta anômala com centenas de hosts ativos simulados.

* **Evidência do MAC Único (Spoofing/Mirroring):** Independentemente do IP listado na faixa (ex: `192.168.4.14` ao `192.168.5.254`), 100% dos hosts retornam exatamente o mesmo endereço físico de hardware: **`DC:2C:6E:A4:2D:A7`**, associado a ativos corporativos **MikroTik (Routerboard)**.
* **Mecanismo de Captive portal e interceptação Proxy:** A varredura identificou persistência generalizada de respostas nas portas de transporte de dados (`80`,`443`,`3128`,`8080`). O diagnóstico detalhado nas portas revelou o código e status **`HTTP/1.1 301 Hostpost`**.

**Conclusão da Fase A:** O gateway utiliza uma política agressiva de *Mirroring* e Defesa Ativa. Ele responde por toos os IPs da sub-rede para ofuscar o ambiente real e interceptar requisições HTTP antes a autenticação no servidor central.

### Fase B: Análise de Pacotes Brutos e Anomalia TLS (Wireshark)
A captura de tráfego em tempo real na interface física `wlp3s0` expôs tentativas ativas de manipulação de tráfego seguro por parte de infraestrutura da rede:
* **Injeção de Pacotes TCP RST (Reset):** Durante tentativas legítimas de estabelecimento de sessões seguras na porta `443` (HTTPS), o tráfego registrou pacotes forçados com a flag `[RST]`. Isso indica a derrubada  intencional de conexções criptografadas de ponta a ponta (técnica correlata a ataques de *Man-in-the-Middle* para  forçar o downgrade de segurança).
* **Quebra de Handshake TLS (x.509):** O log registrou pacotes do tipo `TLSv1.2 Alert (Level: Warning, Description: Close Notify)` imediatamente após o envio do pacote inicial `Client Hello`. A infraestrutura tentou forçar a troca do certificado digital por uma autoridade de certificação desconhecida e inválida, impedindo a inicialização de túneis seguros e aplicativos de VPN de host.
* **Bloqueio Administrativo (ICMP Tipo 3, Cóigo 13):** O roteador (`192.168.4.1`) respondeu sistematicamente às requisições de host com pacotes contendo a mensagem **`Destination unreachable (Network administratively prohibited)`**, bloqueando o acesso a repositórios de segurança externos e pacotes de atualização.

---

## 4. Engenharia de Defesa e Resposta a Incidentes (Hardening)
Diante a hostilidade do perímetro da rede, foram aplicadas camadas estritas de proteção no host local Linux para neutralizar quaisquer tentativas de exploração ou vazamentos de dados:

### 1. Fortificação de Firewall Host-Based (Netfilter/UFW)
A aplicação de políticas rigorosas de filtragem isolou o sistema de qualquer interação indesejada oriunda da rede local:
```bash
sudo ufw reset
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable
```
*Garantia técnica: O host passou a rejeitar  silenciosamente qualquer pacote de entrada vindo a rede (`ìncoming: deny`), mantendo apenas conexões de saída iniciadas pelo próprio usuário.*

### 2. Bloqueio Estrito de Protocolos de compartilhamento (Samba/NetBIOS)
Para mitigar a vulnerabilidade de exposição de portas locais em ambientes públicos, foi forçado o bloqueio das portas de comprtilhamento de arquivos do sistema (`139`e `445` TCP/UDP):
```bash
sudo ufw deny proto tcp from any to any port 139,445
sudo ufw reload
```

### 3. Isolamento Total por Migração de Vetor (Hostpost Celular)
Como a infraestrutura realizava inspeção profunda e quebra de TLS inviabilizando o uso seguro de VPNs, a conexão com o Wi-Fi do estabelecimento foi abortada. O tráfego foi migrado integralmente para uma interface de ancoragem móvel (Hostpost 4G/5G privado), estabelecendo um canal de comunicação unívoro, criptografado na origem livre dos riscos do gateway local.

---

## 4. Conclusão e Lições Aprendidas
Este incidente prático demonstrou que a segurança em ambientes Públicos não deve depender da confiança na infraestrutura de terceiros (*Zero Trust*). A análise detalhada e pacotes provou ser indispensável para diagnosticar por que ferramentas automatizadas de segurança (como VPNs) falham sob regras agressivas de roteadores corporativos.

A aplicação imediata de regras e firewall locais de host (*Hardening*) e a mudança de vetor e rede garantiram a integridade absoluta dos dados e do ativo de TI sob um cenário de rede hostil. 

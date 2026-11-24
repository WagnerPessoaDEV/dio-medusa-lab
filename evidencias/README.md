# Evidências

Capturas locais dos três cenários executados no laboratório host-only. As capturas foram sanitizadas: senhas e cookie de sessão aparecem como `[REDACTED]`.

- [FTP — Medusa](01-ftp-medusa.png)
- [SMB password spray — Medusa](02-smb-medusa.png)
- [DVWA web-form — Medusa](03-dvwa-medusa.png)

O detalhamento dos resultados está em [resultado-pratico.md](resultado-pratico.md). As saídas temporárias do terminal foram removidas após a execução. O primeiro teste web em `/dvwa/login.php` encontrou HTTP 302; o teste final usou a página brute do DVWA e encontrou uma combinação válida. A configuração foi restaurada para `high` e a sessão temporária foi encerrada.

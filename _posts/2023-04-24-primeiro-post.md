---
layout: post
title: Sometimes you gotta close a door to open a Windows.
date: 2023-04-24 17:30:50 +0800
categories: [Red Team, Active Directory]
tags: [Red Team]
---

teste
teste2


Como sabemos, durante um teste de intrusão.

Como um pentester, diferentes cenários para exploração tornam-se rotineiros. E, por consequência, cenários peculiares também entram na lista do que um pentester irá enfrentar durante seu teste.

Durante um teste de intrusão, eu e meu colega de trabalho, [oppsec](https://www.linkedin.com/in/dansdm), nos deparamos com um cenário específico: conseguimos acesso de administrador de domínio sem antes ter comprometido um único Windows. Sem movimentação lateral, sem CVEs, sem acesso inicial, sem exfiltração de dados. Utilizamos unicamente o nosso sistema para obtermos acesso de DA.

## Reconhecimento.

Nos encontrávamos em uma rede que continha mais de 15 faixas de IPs, e geralmente com uma subnet de /23 ou /24 (em tornos de 512 ou 256 ativos por range). Como processo inicial, nada melhor que uma boa enumeração da infraestrutura. Para isso, utilizamos o clássico NMAP para varrermos endereços com portas comumente encontradas, tais como: 22 (SSH), 80 (HTTP), 443 (HTTPS), 445 (SMB), 3389 (RDP) e 5985 (WinRM).

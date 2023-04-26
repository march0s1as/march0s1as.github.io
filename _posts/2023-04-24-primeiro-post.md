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

Durante um teste de intrusão, eu e meu colega de trabalho, [oppsec](https://www.linkedin.com/in/dansdm/), nos deparamos com um cenário específico: conseguimos acesso de administrador de domínio sem antes ter comprometido um único Windows. Sem movimentação lateral, sem CVEs, sem acesso inicial, sem exfiltração de dados. Utilizamos unicamente o nosso sistema para obtermos acesso de DA.

## Reconhecimento.

Nos encontrávamos em uma rede que continha mais de 15 faixas de IPs, e geralmente com uma subnet de /23 ou /24 (em tornos de 512 ou 256 ativos por range). Como processo inicial, nada melhor que uma boa enumeração da infraestrutura. Para isso, utilizamos o clássico NMAP para varrermos endereços com portas comumente encontradas, tais como: 22 (SSH), 80 (HTTP), 443 (HTTPS), 445 (SMB), 3389 (RDP) e 5985 (WinRM).

Durante o processo de reconhecimento, foram encontrados alguns serviços web que foram de grande valia para a exploração. Dentre eles, um serviço de HelpDesk que aceitava autenticação de usuários do Active Directory. Como já tínhamos acesso a um usuário (não privilegiado) antes, conseguimos nos autenticar no serviço como usuário comum.

Após acessarmos tal aplicação, nos deparamos com diversas opções que serviam como auxílio aos colaboradores da companhia. Como por exemplo, descrevia como acessar um sistema específico, como solicitar ajuda em um problema técnico, entre outros. Decidimos por analisar todos os tópicos contidos na documentação e, ao ver um chamado em aberto, acabamos por achar uma aplicação conveniente para nossa situação.

Se tratava de um sistema de gerenciamento de assinaturas de e-mail feito em PHP, que por sua vez, poderia ter diversas vulnerabilidades que nos renderiam algum acesso ou informações privilegiadas sobre algum usuário da rede.

## SQL Injection.

Na aplicação encontrada, existiam alguns funções como: selecionar um funcionário específico da rede, modelo especifico da assinatura e afins. Visto isso, fizemos uma seleção aleatória e clicamos no botão de “Processar” para que a assinatura fosse gerada.

Com a assinatura montada pela aplicação, notamos que existiam parâmetros repassados via GET e que, por sua vez, poderiam estar vulneráveis a ataques de injeção de SQL. E, para nossa felicidade (rs), ao repassarmos uma aspas simples a um dos parâmetros, o seguinte erro aparecia:

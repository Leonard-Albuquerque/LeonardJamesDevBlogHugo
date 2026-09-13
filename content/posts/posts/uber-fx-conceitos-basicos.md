---
title: "Go: Uber Fx — Conceitos Básicos e Injeção de Dependência"
date: 2026-09-13T17:05:37-03:00
tags: ["golang", "estudos", "uber-fx", "arquitetura"]
author: "Leonard James"
showToc: true
draft: false
description: "Guia prático para desmistificar a Injeção de Dependência e o ciclo de vida em Go utilizando o Uber Fx."
---

## Entendendo o problema

Quando começamos a construir aplicações em Go, é comum ver o arquivo `main.go` crescer rapidamente à medida que adicionamos novos serviços, repositórios e controladores. Em pouco tempo, nos deparamos com o chamado **wiring manual**: ter que instanciar cada dependência na ordem exata para conseguir subir a aplicação.

O **Uber Fx** surge como uma ferramenta para resolver essa dor, automatizando a montagem do grafo de dependências e gerenciando o ciclo de vida da aplicação de forma limpa.

Neste post, vamos passar pela base de interfaces, entender a Inversão de Dependência e ver como o Uber Fx funciona na prática.

---

## Parte 1 — A base: entendendo interfaces em Go

Antes de falar de Injeção de Dependência (DI) ou Uber Fx, precisamos entender como Go lida com variáveis e interfaces. É comum a cabeça dar um nó na hora de ler um código como este:

```go
aluno := &Aluno{nome: "Maria", idade: 21}
var usuario Usuario = aluno // aluno "vira" um Usuario aqui
```

### Como ler isso sem travar?

> *"Crie uma variável chamada `usuario`, que aceita qualquer coisa do tipo `Usuario` (Interface), e guarde o `aluno` dentro dela."*

### A analogia do crachá

Pense na interface `Usuario` como um crachá de acesso corporativo. A Maria (`aluno`) não deixou de ser Maria. Ela apenas vestiu o crachá de `Usuario`. Quem olha para ela agora só se importa com o que o crachá permite fazer (ex: `Nome()` e `Idade()`), sem precisar saber se por baixo ela é um `Aluno`, `Professor` ou `Funcionario`. 

O lado direito do `=` é sempre o dado real que já existe, e o lado esquerdo é a "fantasia" ou contrato que você escolheu dar a ele.

---

## Parte 2 — Por que não acessar a struct direto? (Inversão de Dependência)

Você pode se perguntar: *"Se eu já tenho o `Aluno`, por que criar uma interface só para chamar o nome dele? Não é retrabalho?"*

O segredo é: **interfaces não foram feitas para quem CRIA o dado, mas para as outras partes do sistema que CONSOMEM esse dado.**

Se uma função do seu sistema pede diretamente o `Aluno`:

```go
func EnviarEmailBoasVindas(a Aluno) { ... }
```

Ela virou refém. Se amanhã surgir um `Professor`, você tem que reescrever a função ou duplicá-la.

Agora, se você depende de uma **Interface**:

```go
// Contrato de alto nível (A Regra): "Preciso de algo que busque usuários"
type RepositorioDeUsuario interface {
    BuscarUsuario() Usuario
}

// Implementação de baixo nível (O Detalhe): A "cozinha" que faz o trabalho sujo
type RepositorioMemoria struct{}
func (r RepositorioMemoria) BuscarUsuario() Usuario {
    return &Aluno{nome: "Maria"}
}
```

* **Alto nível vs. Baixo nível:** "Alto nível" é a regra de negócio (a intenção do sistema). "Baixo nível" é o detalhe técnico de como a coisa é feita por baixo dos panos (banco de dados, memória, leitura de arquivo).
* **A vantagem (Inversão de Dependência):** Se amanhã você trocar o `RepositorioMemoria` por um `RepositorioPostgres`, nenhuma outra regra de negócio do seu sistema precisa mudar. O contrato continua o mesmo.

---

## Parte 3 — O que o Uber Fx resolve na prática?

Sem o Fx, conforme seu app cresce, a função `main()` vira um pesadelo de "criar isso, depois aquilo, depois injetar aqui, depois ali, na ordem exata". Chamamos isso de **wiring manual**.

Veja a diferença:

### O pesadelo manual (sem Fx)

```go
db := banco.NovoPostgres("url...")
repoUsuario := repositorio.NovoUsuario(db) // Depende do db
repoEmail := repositorio.NovoEmail(db)     // Depende do db
servicoAuth := servico.NovoAuth(repoUsuario, repoEmail) // Depende dos repos
controllerAuth := http.NovoAuthController(servicoAuth)  // Depende do serviço
servidor := http.NovoServidor(controllerAuth)
servidor.Iniciar()
```

Se você errar a ordem dessas linhas, o código nem compila ou quebra em tempo de execução. Você precisa passar tudo na unha.

### A solução com Uber Fx

O Fx funciona como um **garçom inteligente**. Você apenas entrega um "livro de receitas" para ele (construtores) e diz o que quer receber no final. Ele descobre sozinho a ordem de preparo:

* Monta as peças na ordem exata baseado no que cada uma precisa.
* Evita variáveis globais.
* Desliga tudo de forma organizada (Graceful Shutdown).

---

## Parte 4 — Instalando e o primeiro "Hello World"

Instalação do pacote:

```bash
go get go.uber.org/fx
```

Código básico:

```go
package main

import "go.uber.org/fx"

func main() {
    fx.New().Run()
}
```

* `fx.New(...)` cria a aplicação Fx.
* `.Run()` liga tudo e **bloqueia** o programa, esperando um sinal de parada (como `Ctrl+C`).

---

## Parte 5 — Ensinando o Fx a montar suas peças (Provide & Invoke)

O Fx **não aceita instâncias soltas** (como `RepositorioMemoria{}`). Ele precisa de **Funções Construtoras** (fábricas).

```go
package main

import (
    "fmt"
    "go.uber.org/fx"
)

// 1. Interfaces e Structs
type RepositorioDeUsuario interface { BuscarUsuario() string }
type RepositorioMemoria struct{}
func (r RepositorioMemoria) BuscarUsuario() string { return "Maria" }

// 2. A FÁBRICA (Essencial para o Fx)
func NovoRepositorio() RepositorioDeUsuario {
    return RepositorioMemoria{}
}

// 3. UMA CAMADA DE SERVIÇO (Para ver o Fx trabalhando)
type ServicoNotificacao struct {
    repo RepositorioDeUsuario
}
func NovoServico(r RepositorioDeUsuario) *ServicoNotificacao {
    return &ServicoNotificacao{repo: r}
}

func main() {
    fx.New(
        // O LIVRO DE RECEITAS:
        fx.Provide(
            NovoRepositorio,
            NovoServico,
        ),
        // O PEDIDO DO CLIENTE:
        fx.Invoke(func(servico *ServicoNotificacao) {
            usuario := servico.repo.BuscarUsuario()
            fmt.Println("Usuário encontrado:", usuario)
        }),
    ).Run()
}
```

### O que o Fx fez aqui?

1. **`Provide`:** Ele anotou: *"Se pedirem um Repositorio, eu chamo `NovoRepositorio`. Se pedirem um Servico, eu chamo `NovoServico` (mas a fábrica de serviço precisa do Repositorio primeiro!)"*.
2. **`Invoke`:** Você pediu que ele executasse uma função que exige um `*ServicoNotificacao`.
3. O Fx leu o grafo de dependências: executou `NovoRepositorio`, pegou o resultado, injetou automaticamente no `NovoServico`, e entregou o serviço pronto para a função no `Invoke`.

---

## Parte 6 — Ciclo de vida: o desligamento organizado (Graceful Shutdown)

O Fx gerencia recursos que precisam iniciar e parar corretamente (como servidores HTTP ou conexões de banco de dados) através do `fx.Lifecycle`.

### O que significa desligar de forma organizada?

Imagine um restaurante. Se acabar o expediente e você simplesmente puxar a chave de força da luz (modo manual com `Ctrl+C`), os clientes comem no escuro, saem sem pagar e a comida no fogão queima. Na programação, isso gera dados corrompidos e requisições caindo com erro `502 Bad Gateway`.

O **Graceful Shutdown** do Fx faz o oposto:

1. Tranca a porta para novos clientes (para de aceitar novas requisições).
2. Espera quem já está comendo terminar de comer e pagar (aguarda requisições em andamento).
3. Limpa a cozinha na ordem inversa (fecha conexões com banco de dados).
4. Desliga as luzes de forma segura.

### Exemplo de implementação

```go
func NovoServidorGin(lc fx.Lifecycle) *gin.Engine {
    router := gin.Default()
    servidor := &http.Server{Addr: ":8080", Handler: router}

    lc.Append(fx.Hook{
        OnStart: func(ctx context.Context) error {
            // GO na frente é vital! Sem ele, o Fx trava esperando o servidor parar e nunca termina de ligar.
            go servidor.ListenAndServe() 
            fmt.Println("Servidor no ar em :8080")
            return nil
        },
        OnStop: func(ctx context.Context) error {
            fmt.Println("Desligando servidor suavemente...")
            // Para de aceitar novas requisições, mas termina as em andamento
            return servidor.Shutdown(ctx)
        },
    })

    return router
}
```

* **`OnStart`:** Roda ao iniciar a aplicação. Lembre-se sempre de colocar `go` em chamadas bloqueantes.
* **`OnStop`:** Roda quando a aplicação recebe um sinal de encerramento (`Ctrl+C`). O Fx chama os hooks de `OnStop` na **ordem inversa** de dependência, evitando fechar um banco de dados enquanto o servidor HTTP ainda tenta consultá-lo.

---

## Parte 7 — Glossário rápido

| Termo | Tradução Mental |
|---|---|
| `fx.Provide(fn)` | *"Toma essa receita. Só fabrique se alguém pedir em um Invoke ou outro Provide."* |
| `fx.Invoke(fn)` | *"Execute isso AGORA, e resolva os parâmetros que eu precisar."* |
| `lc.Append(fx.Hook{...})` | Registra as funções exatas de ligar e desligar recursos. |
| `fx.Lifecycle` | *"O livro de ocorrências da portaria. Deixa agendar ações para o OnStart e OnStop."* |
| `.Run()` | *"Ligue tudo, resolva o grafo de dependências e bloqueie a execução aguardando o sinal de desligar."* |

---

## Pra fechar

Tudo isso é baseado no que venho estudando sobre arquitetura e desenvolvimento em Go. Sou um dev júnior com muito aprendizado pela frente, e acho fundamental entender o que acontece por baixo dos panos antes de adotar qualquer framework ou biblioteca.

Espero que esse guia ajude a desmistificar a injeção de dependência com Uber Fx.

Sugestões, correções ou ideias para complementar são super bem-vindas. Meu e-mail está na home.

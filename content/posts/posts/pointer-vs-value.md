---

title: "Go: Structs como Valor ou Ponteiro (sem dor emocional)"
date: 2024-05-24T11:30:03-03:00
tags: ["golang", "estudos"]
author: "Leonard James"
showToc: true
draft: false
description: "Guia rápido: quando usar structs como value ou pointer em Go?"
----------------------------------------------------------------------------

## Let’s Go 🚀

Pra começar em Go, **structs podem ser valores ou ponteiros**.
Isso é bem simples, mas existem alguns pormenores quando falamos de **mudar alguma coisa** nessas structs.

Se você já escreveu código assim:

```go
for _, o := range orders {
    o.Status = "shipped"
}
```

Você provavelmente notou que **não houve alteração alguma** nesses dados.

Resolvi escrever esse artigo exatamente pra falar sobre isso, traduzindo boa parte do que aprendi lendo o post do Preslav Rachev:

👉 [https://preslav.me/2026/01/08/golang-structs-vs-pointers-pointer-first/?utm_source=christophberger&utm_medium=email&utm_campaign=2026-01-11-2026-and-go-126](https://preslav.me/2026/01/08/golang-structs-vs-pointers-pointer-first/?utm_source=christophberger&utm_medium=email&utm_campaign=2026-01-11-2026-and-go-126)

**Quando usar ponteiros? Quando usar valores? E por que isso é relevante no longo prazo?**

---

## Antes de mais nada

1. Structs são valores
2. `range` copia o valor
3. Mutar a cópia **não altera o original**
4. O compilador **não avisa**
5. **Resultado**: dados iguais, bug silencioso 😬

### Exemplo rápido

```go
type Order struct {
    ID     string
    Status string
}

// Slice de values
orders := []Order{
    {ID: "1", Status: "pending"},
}

for _, o := range orders {
    o.Status = "shipped" // muda só a cópia
}
```

O código roda normalmente, mas o resultado final do `Status` continua sendo **"pending"**.

E cá entre nós: em um sistema grande isso vira um bug chato, difícil de achar, que costuma render vários `fmt.Printf()` espalhados pelo código e uma boa dose de paciência.

---

## A ideia de *Pointer‑First*

Em Go, dá pra se nortear por uma premissa bem prática: **Pointer‑First**.

Ou seja: se uma struct tem:

* **estado** ("happy", "sad")
* **identidade** ("creator", "affiliate")
* **ciclo de vida** (`init`, `execute`, `close`)

👉 trate como **ponteiro desde o início**.

O motivo é simples:

> **O custo cognitivo de errar usando valores é maior do que o custo de usar ponteiros.**

### Exemplo

No caso de um `Close()`, você quer garantir que está fechando **a conexão real**, não uma cópia fantasma do objeto.

---

## Como decidir em 10 segundos ⏱️

Faça essas perguntas:

1. Muda ao longo do tempo?
2. Representa algo do mundo real?
3. Vai ser alterado ao passar por várias funções?
4. Tem métodos que modificam seus campos?

👉 Se a resposta for **sim**: **Ponteiro**
👉 Se for **não**: **Value**

---

## Então… quando usar valores?

Valores existem por um motivo e são ótimos quando lidamos com coisas **imutáveis** ou puramente **descritivas**, como:

* configurações
* inputs
* opções
* enums

### Exemplos

```go
type Config struct {
    Timeout time.Duration
    Retries int
}
```

```go
type CreateUserInput struct {
    Email string
    Name  string
}
```

```go
type Options struct {
    DryRun bool
    Debug  bool
}
```

### E na hora de usar?

Isso mesmo… **ponteiro** 😄

```go
func NewClient(cfg Config) *Client {
    ...
}
```

---

## E quanto à performance?

Um argumento comum contra ponteiros é performance:

> “Struct como valor é mais rápido.”
> “Ponteiros causam mais GC.”
> “Copiar struct é caro.”

Na prática, **para a maioria dos sistemas reais**, isso é quase sempre **irrelevante**.

Em Go, a banda costuma tocar assim (bem resumido):

* copiar structs **pequenas** é barato
* passar ponteiros também tem custo
* o compilador otimiza agressivamente
* o impacto real geralmente é **imperceptível**

👉 **Performance raramente é o fator decisivo aqui**.

---

## O custo invisível dos valores fantasmas

Por outro lado, o custo de **não** lidar com valores fantasmas paga fácil o uso de ponteiros.

Eles ajudam a evitar:

* bugs silenciosos
* decisões tardias de design
* refactors grandes

Um `[]Order` que depois vira `[]*Order` dói **muito mais** do que começar certo.

Se você já precisou mexer na cadeia hierárquica inteira do código por causa disso… você sabe a dor 😅

---

## Quando performance *pode* importar

Existem exceções — e elas costumam ser bem claras:

* structs **muito grandes**
* código dentro de **loops muito quentes**
* sistemas de **altíssimo throughput**

Nesses casos:

* **meça**
* **compare**
* **decida conscientemente**

---

## Regra prática

> **Escolha primeiro o modelo mental correto.**
> **Otimize depois — se (e somente se) precisar.**

---

## Resumão 🧠

Para 95% dos sistemas:

* clareza > micro‑performance
* previsibilidade > suposições
* pointer‑first evita mais problemas do que cria
* use value só se a struct for pequenininha e imutável

---

## Pra fechar

Tudo isso é baseado no que estudei até agora. Sou um dev júnior com muito aprendizado pela frente, e não vejo problema nenhum em aprender algo novo — mesmo quando isso mostra que uma crença antiga não era bem uma verdade absoluta.

Espero que isso seja útil pra quem estiver lendo 🙂

Sugestões, correções ou ideias pra complementar são super bem‑vindas.
Meu e‑mail está na home. 😄

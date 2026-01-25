---
title: "Bug no iOS Safari: Ao digitar, a tela fica com um extravasamento na parte inferior (scroll + teclado)"
date: 2024-06-02T10:15:00-03:00
tags: ["frontend", "ios", "safari", "css", "mobile"]
author: "Leonard James"
showToc: true
draft: false
description: "Como resolvi o bug clássico do iOS Safari que duplica inputs fixos ou gera uma seção fantasma em chats quando o teclado está aberto."
---

Durante o desenvolvimento da tela de chat (estilo WhatsApp/Telegram), enfrentei um **bug clássico do iOS Safari** que causa **duplicação visual do layout**, especialmente do **campo de digitar mensagem**, quando o usuário faz scroll com o teclado aberto.

Esse problema aparece principalmente em aplicações mobile com:

- `position: fixed` ou layouts que simulam fixo
- inputs presos ao rodapé
- containers com `100vh` ou variações
- listas com scroll interno (ex: chat)

---

## 🧨 O problema

No iOS Safari, quando o teclado virtual abre:

- O navegador **recalcula o viewport de forma incorreta**
- O `100vh` deixa de representar a área visível real
- O Safari **cria um espaço extra invisível**
- Ao puxar a tela (scroll elástico), o layout **parece duplicar**

No meu caso específico:

- O campo de digitar mensagem aparecia **duas vezes**
- O bug só acontecia no **iPhone (Safari)**
- Desktop e Android funcionavam normalmente

---

## 📚 Referências (problema conhecido)

Esse comportamento não é bug do React nem do CSS — é **do motor do Safari**:

- StackOverflow: Sticky input criando espaço em branco
- https://stackoverflow.com/questions/54408491/ios-safari-sticky-input-at-bottom-unwanted-white-space

- Medium: `position: fixed` no Safari
- https://medium.com/@im_rahul/safari-and-position-fixed-978122be5f29

- Reddit / WebDev: teclado do iOS quebrando layout
- https://www.reddit.com/r/webdev/comments/xaksu6/on_ios_safari_whenever_the_keyboard_opens_up_for/

---

## 🧠 Diagnóstico

Depois de testar várias abordagens (CSS puro, `vh`, `svh`, `dvh`, re-render condicional), percebi que:

> O bug só acontece **quando o input está focado** e o usuário **tenta scrollar a lista**.

Ou seja:

- Teclado aberto
- Input focado
- Scroll no container

➡️ Safari entra em estado inconsistente

---

## ✅ Solução Sugerida

Ao invés de tentar brigar com o cálculo de viewport do Safari, a solução foi **fechar o teclado automaticamente quando o usuário tenta scrollar o chat**.

### Estratégia

- Detectar `touchmove` no container do chat
- Se existir um input focado → remover o foco (`blur()`)
- O teclado fecha
- O Safari volta ao estado normal
- O layout para de duplicar ou gerar a extensão mal calculada do body 

---

## 🛠️ Implementação

### 1️⃣ Referência para o container do chat

```tsx
const containerRef = useRef<HTMLDivElement>(null);
```

---

### 2️⃣ Effect para fechar o teclado ao scrollar

```tsx
useEffect(() => {
  const handleScroll = () => {
    if (document.activeElement instanceof HTMLInputElement) {
      document.activeElement.blur();
    }
  };

  if (containerRef.current) {
    containerRef.current.addEventListener('touchmove', handleScroll, {
      passive: true,
    });

    return () => {
      containerRef.current?.removeEventListener('touchmove', handleScroll);
    };
  }
}, []);
```

---

### 3️⃣ Container principal do ChatMessage

```tsx
<div
  ref={containerRef}
  className="flex flex-col w-full h-[92vh] md:h-full bg-zinc-950 text-white"
>
```

📌 Observações importantes:

- O listener é aplicado **somente no container do chat**
- Não interfere em outros scrolls da aplicação
- Usa `passive: true` (boa prática para performance)

---

## 🎯 Resultado

- ❌ Campo de digitar não duplica mais
- ❌ Sem espaço em branco fantasma
- ❌ Sem layout quebrando no scroll elástico
- ✅ Comportamento natural para o usuário
- ✅ Funciona apenas no mobile (onde faz sentido)

---

**Adendo :**

A maioria dos sites que eu peguei, apresentavam esse problema no SAFARI, e basicamente, não é como se dificultasse a usuabilidade, mas o UX fica meio estranho se for um usuário mais cricri que gosta de fuçar. Bem, espero que isso resolva o problema para  quem quer que  chegue a ler esse artigo. 

---


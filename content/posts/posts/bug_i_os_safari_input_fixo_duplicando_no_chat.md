# Bug no iOS Safari: input fixo duplicando no chat (scroll + teclado)

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

## ✅ Solução adotada (simples e eficiente)

Ao invés de tentar brigar com o cálculo de viewport do Safari, a solução foi **fechar o teclado automaticamente quando o usuário tenta scrollar o chat**.

### Estratégia

- Detectar `touchmove` no container do chat
- Se existir um input focado → remover o foco (`blur()`)
- O teclado fecha
- O Safari volta ao estado normal
- O layout para de duplicar

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

## 🧩 Conclusão

Esse é um daqueles bugs clássicos onde:

- O problema **não é seu código**
- O problema **não é o framework**
- O problema é o navegador

Em vez de forçar CSS complexo ou gambiarras com viewport, **fechar o teclado no scroll** se mostrou a solução mais estável e previsível para chats no iOS Safari.

Se você está construindo um chat, feed ou formulário com input fixo no rodapé:

👉 **considere esse padrão desde o início**.

---

Se quiser, no próximo post posso documentar:
- diferenças entre `vh`, `svh`, `dvh`
- quando usar `position: fixed` vs `sticky`
- arquitetura ideal de chat para mobile


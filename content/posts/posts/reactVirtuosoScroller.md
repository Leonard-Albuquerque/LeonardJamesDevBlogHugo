---
title: "Next.js + React-Virtuoso: como eliminar scroll duplo e bugs de scroll no mobile"
date: 2026-01-30T14:40:00-03:00
tags: ["nextjs", "react", "react-virtuoso", "frontend", "mobile", "scroll"]
author: "Leonard James"
showToc: true
draft: false
description: "Como resolvi o problema clássico de scroll duplo (body + Virtuoso) no Next.js App Router, garantindo scroll único, performance correta e melhor UX no mobile."
---

Durante o desenvolvimento do **feed autenticado** e da tela **/board** da aplicação, me deparei com um problema extremamente comum quando se usa **React-Virtuoso** em conjunto com **Next.js (App Router)**:

> **scroll duplo** — o `body` da página continua scrollando enquanto o Virtuoso também controla seu próprio scroll.

No desktop isso já é incômodo.  
No **mobile**, especialmente no iOS, o problema fica bem mais evidente.

---

## **O problema ...**

O cenário era o seguinte:

- Aplicação em **Next.js (App Router)**
- Um **layout global** envolvendo toda a aplicação
- Algumas telas usando **React-Virtuoso**
- Outras telas com scroll normal do body

Resultado:

- Scroll do `body` ativo
- Scroll interno do Virtuoso ativo
- Dois scrolls competindo entre si
- Mobile com bounce, scroll elástico e sensação de "layout quebrado"

---

## 🚫 O que gera mais problemas do que resolve?

Em primeira tentativa eu tentei ir por um caminho como : 

```css
body {
  overflow: hidden;
}
```

Adicionar isso no layout global ou tentar passar css injetado direto nos componentes.
**Não faça isso.**

Por quê? Porque você o bloqueio do scroll vai estar na  **aplicação inteira**, o que quebra páginas que usam scroll normal do body. 


ps: O que no meu caso ( app de rede social ) não era nem de longe o comportamento que eu precisava ... 

---

## 📱 Por que no mobile é pior?

No mobile:

- O navegador já aplica **overscroll** (aquele bounce elástico)
- O viewport é recalculado constantemente
- O scroll do `body` interfere diretamente no scroll interno
- O usuário perde previsibilidade ao rolar a tela
- Em alguns casos, o scroll parecia "puxar" a tela inteira, mesmo com o Virtuoso no meio da hierarquia

No iOS especificamente, esse comportamento é ainda mais pronunciado devido ao `WebkitOverflowScrolling`.

---

## ✅ A solução

### Passo 1: Criar um layout dedicado para páginas com Virtuoso

Crie uma pasta `(virtuoso-routes)` dentro de `app/` e um layout específico:

```
app/
├── layout.tsx (layout global)
├── (virtuoso-routes)/
│   ├── layout.tsx (layout apenas para rotas com Virtuoso)
│   ├── page.tsx (feed)
│   └── board/
│       └── page.tsx
└── other-page/
    └── page.tsx (scroll normal)
```

### Passo 2: Layout para páginas com Virtuoso

`app/(virtuoso-routes)/layout.tsx`:

```tsx
"use client";

import { useEffect } from "react";

export default function VirtuosoLayout({ children }) {
  useEffect(() => {
    // Salva os valores originais para restaurar depois
    const originalOverflow = document.body.style.overflow;
    const originalHeight = document.body.style.height;

    // Bloqueia o scroll do body
    document.body.style.overflow = "hidden";
    document.body.style.height = "100vh";

    // Cleanup: restaura os valores quando o layout é desmontado
    return () => {
      document.body.style.overflow = originalOverflow;
      document.body.style.height = originalHeight;
    };
  }, []);

  return (
    <div className="h-screen overflow-hidden">
      {children}
    </div>
  );
}
```

### Passo 3: Configurar o Layout  corretamente

Dentro da página (ex: `app/dashboard/page.tsx`):

```tsx
export default function HomePage(){
    return (
        <VirtuosoLayout>
            <AuthenticatedFeed/> // Seu componente que precisa do Scroll travado
        </VirtuosoLayout>
    );

}
```

**Ponto importante:**

-- Isso serve também quando tem um header e um footer ( no meu caso um header e um float menu ).


-- O resultado dessa abordagem é garantir que apenas o que possui o virtuoso diretamente naquela page poderá scrollar.


-- Saliento que é ˜apenas o que está diretamente naquela pagina˜, por que se caso houver um modal abrindo acima daquele body, o scroll do modal irá funcionar normalmente ( que é o esperado ).

---

## 🔗 Base bibliografica

- [React-Virtuoso Documentation](https://virtuoso.dev/)
- [CSS overscrollBehavior](https://developer.mozilla.org/en-US/docs/Web/CSS/overscroll-behavior)

---

### Dica/Resumo :
  Controle de scroll é , via de regra, mais facil de controlar usando layouts granulados, assim fica isolado a lógica para onde você precisa. No meu contexto, eu tenho uma rede social, e só podia efetuar o travamento do scroll em telas onde o virtuoso fazia o gerenciamento de scroll.

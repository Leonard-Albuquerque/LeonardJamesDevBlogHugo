---
title: "Bloqueando scroll do react-virtuoso quando um modal está aberto"
date: 2024-06-05T09:40:00-03:00
tags: ["react", "nextjs", "ux", "react-virtuoso", "frontend"]
author: "Leonard James"
showToc: true
draft: false
description: "Estratégia simples usando pointer-events para bloquear o scroll do react-virtuoso quando um modal está ativo."
---

## Problema

Quando usando `react-virtuoso` para renderizar listas virtualizadas em Next.js, abrir um modal de comentários permite que o usuário continue scrollando a lista por baixo do modal. 

#### No meu caso gerou um bug no meu feed, mais pra um efeito colateral.

Uma observação importante é que meu bloco do virtuoso e dos comentários foram feitos separados, o feed pertece ao virtuoso mas os comentários não. O por que isso é relevante? Simples, se a seção de comentários for acoplada dentro do virtoso, a própria lib faz a tarefa de lidar com o bloqueio do scrol no layer mais externo.

## Solução

A solução é usar a propriedade CSS `pointerEvents` combinada com um estado global ou prop transportada do component pai até o filho/neto que rastreia se o modal está aberto.

## Implementação

### 1. No componente pai (Feed com Virtuoso)

```typescript
const { profile, commentOnModal } = useBaseContext();

<Virtuoso<IPost>
    style={{
      height: "100vh",
      width: "100%",
      pointerEvents: !commentOnModal ? "none" : "auto"
    }}
    data={feed}
    itemContent={(
        _index: number,
        item: IPost
    ): React.ReactElement => (
        <FeedPost
            post={item}
            profileId={profile?.id ?? ""}
            onFeedUpdate={setFeed}
        />
    )}
/>
```

### 2. No componente do Modal

```typescript
<div
    style={{ pointerEvents: !commentOnModal ? "auto" : "none" }}
    className="fixed inset-0 z-50 flex items-end justify-center">
    {/* Conteúdo do modal */}
</div>
```

### 3. Gerenciar o estado global (mais simples de exemplificar aqui)

No seu `BaseContext`, certifique-se de ter:

```typescript
const [commentOnModal, setCommentOnModal] = useState(false);

// Ao abrir o modal de comentários
setCommentOnModal(false); // desabilita pointer events do Virtuoso

// Ao fechar o modal
setCommentOnModal(true); // reabilita pointer events do Virtuoso
```

### 4. Bloquear o scroll do body (opcional)

Para máxima compatibilidade, adicione também ao `CommentsModal`:

```typescript
useEffect(() => {
  document.body.style.overflow = "hidden";
  return () => {
    document.body.style.overflow = "";
  };
}, []);
```

## Como funciona

- **`pointerEvents: "none"`** - Desabilita todos os eventos de mouse/toque no Virtuoso, bloqueando scroll
- **`pointerEvents: "auto"`** - Reabilita eventos de mouse/toque no Virtuoso
- **`commentOnModal`** - Estado booleano que indica se o modal está aberto

Quando `commentOnModal` é `false` (modal aberto):
- Virtuoso: `pointerEvents: "none"` (não responde a scroll)
- Modal: `pointerEvents: "auto"` (responde normalmente)

Quando `commentOnModal` é `true` (modal fechado):
- Virtuoso: `pointerEvents: "auto"` (responde a scroll)
- Modal: `pointerEvents: "none"` (não existe/não renderizado)

## Benefícios

✅ Simples e direto  
✅ Sem listeners de eventos complexos  
✅ Funciona em todos os navegadores  
✅ Sem quebra de funcionalidade (inputs e botões funcionam normalmente)  
✅ Compatível com `react-virtuoso`


E pronto tudo certo, espero que lhe tenha sido útil!
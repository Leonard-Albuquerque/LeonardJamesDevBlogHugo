---
title: "Implementando ACL no frontend com Next.js, Context API e permissões vindas da API"
date: 2026-03-23T10:15:00-03:00
tags: ["frontend", "nextjs", "react", "acl", "auth", "typescript"]
author: "Leonard James"
showToc: true
draft: false
description: "Como implementei um ACL no frontend usando Next.js, React Context e permissões vindas da API para controlar menu, navegação e renderização de módulos."
---

Durante a implementação de uma área autenticada com múltiplos módulos, precisei resolver um problema comum em sistemas administrativos: **o frontend precisava respeitar as permissões reais do usuário**, e não apenas esconder itens visualmente com base em um mock ou numa role fixa.

A ideia era simples:

- o backend autentica o usuário
- o endpoint `/auth/me` devolve `role` e `permissions`
- um contexto base carrega esse perfil
- um provider de ACL transforma essas permissões em regras de acesso
- o layout usa isso para **mostrar, esconder e redirecionar**

No fim, a solução ficou bem limpa e reutilizável.

---

## 🧨 O problema

Antes dessa implementação, o controle de acesso no frontend estava preso a um perfil mockado.

Na prática, isso gerava alguns riscos:

- o menu lateral podia exibir módulos errados
- o usuário podia entrar em uma rota sem ter permissão real
- a interface não refletia o payload devolvido pela API
- o frontend ficava desalinhado com o backend

Ou seja: visualmente parecia existir ACL, mas ele ainda não estava **acoplado ao contexto real de autenticação**.

---

## 🧠 A ideia da solução

Ao invés de espalhar `if`s pela aplicação inteira, a solução foi centralizar a autorização em uma camada única.

A estrutura ficou assim:

- `BaseContext`
- `permissions.ts`
- `acl-context.tsx`
- `app/layout.tsx`
- `dashboard/layout.tsx`

Com isso, o fluxo passa a ser:

1. O usuário faz login
2. O token é salvo
3. O app consulta `/auth/me`
4. O `profile` entra no contexto
5. O ACL converte `permissions[]` em regras utilizáveis
6. O layout filtra os módulos permitidos
7. Se a URL atual não for permitida, o usuário é redirecionado

---

## 📦 Payload esperado da API

O ACL no frontend precisa de um objeto parecido com esse:

```json
{
  "id": 1,
  "super": true,
  "companyId": 1,
  "role": "super_admin",
  "permissions": [
    "users_module",
    "kanban_module",
    "consultation_module",
    "finance_module",
    "internal_chat_module",
    "schedule_module",
    "campaigns_module",
    "whatsapp_module",
    "integrations_module"
  ]
}
```

Os campos que realmente importam para o ACL são:

- `id`
- `companyId`
- `role`
- `permissions`

Sem isso, o frontend não tem como montar o mapa de acesso do usuário.

---

## 🛠️ Implementação

### 1️⃣ Definindo as permissões reconhecidas pelo frontend

O primeiro passo foi criar um catálogo central de permissões.

Isso evita:

- strings soltas pela aplicação
- inconsistência entre módulos
- condicionais repetidas em vários componentes

```ts
export const PERMISSIONS = {
  USERS: "users_module",
  KANBAN: "kanban_module",
  CONSULTATION: "consultation_module",
  FINANCE: "finance_module",
  INTERNAL_CHAT: "internal_chat_module",
  SCHEDULE: "schedule_module",
  CAMPAIGNS: "campaigns_module",
  WHATSAPP: "whatsapp_module",
  INTEGRATIONS: "integrations_module",
  PERMISSIONS: "permissions_module",
} as const;
```

---

### 2️⃣ Criando o registro de módulos

Além da lista de permissões, criei um `MODULE_REGISTRY` para descrever os módulos da interface.

Cada item tem:

- a permissão exigida
- o nome exibido
- a rota
- a lógica para identificar se a rota atual pertence ao módulo

```ts
export const MODULE_REGISTRY: ModuleDefinition[] = [
  {
    permission: null,
    label: "Dashboard",
    href: "/dashboard",
    icon: LayoutDashboard,
    matchPath: (p) => p === "/dashboard",
  },
  {
    permission: PERMISSIONS.KANBAN,
    label: "Gestão de Leads",
    href: "/dashboard/leads",
    icon: Megaphone,
    matchPath: (p) => p.startsWith("/dashboard/leads"),
  },
  {
    permission: PERMISSIONS.INTERNAL_CHAT,
    label: "Chat",
    href: "/dashboard/chat",
    icon: Bell,
    matchPath: (p) => p.startsWith("/dashboard/chat"),
  },
  {
    permission: PERMISSIONS.CONSULTATION,
    label: "Pacientes",
    href: "/dashboard/patients",
    icon: User,
    matchPath: (p) => p.startsWith("/dashboard/patients"),
  },
];
```

📌 Observação importante:

- `permission: null` significa módulo sempre visível
- isso é útil para home, dashboard raiz ou telas neutras

---

### 3️⃣ Carregando o perfil no contexto base

Depois, o `BaseContext` passou a ser o responsável por buscar o perfil autenticado.

Aqui a lógica é:

- verificar se existe token
- chamar `authMeAction()`
- armazenar `profile`
- remover token e redirecionar se receber `401`

```tsx
const findUserProfile = async () => {
  const response: IMeActionResponse = await authMeAction();

  if (response.ok && response.data) {
    setProfile(response.data);
    return;
  }

  if (response.status === 401) {
    await removeAccessToken();
    window.location.replace("/auth");
  }
};
```

Esse `profile` é o combustível do ACL.

---

### 4️⃣ Transformando o profile em regras de ACL

No `ACLProvider`, o contexto de autenticação é lido e convertido para um modelo mais útil para a UI.

A implementação faz quatro coisas:

- valida se a role recebida existe
- filtra permissões desconhecidas
- cria um `Set` para busca rápida
- calcula os módulos liberados

```tsx
export function ACLProvider({ children }: ACLProviderProps) {
  const { profile } = useBaseContext();

  const userAcl = useMemo(() => {
    if (!profile || !isValidRole(profile.role)) {
      return buildFallbackAcl();
    }

    return {
      id: profile.id,
      role: profile.role,
      companyId: profile.companyId,
      permissions: profile.permissions.filter(isValidPermission),
    };
  }, [profile]);

  const permissionSet = useMemo(
    () => new Set<string>(userAcl.permissions),
    [userAcl.permissions]
  );

  const hasPermission = useCallback(
    (permission: Permission) => permissionSet.has(permission),
    [permissionSet]
  );

  const allowedModules = useMemo(
    () =>
      MODULE_REGISTRY.filter(
        (mod) => mod.permission === null || permissionSet.has(mod.permission)
      ),
    [permissionSet]
  );

  const value = useMemo(
    () => ({ userAcl, hasPermission, allowedModules, isReady: Boolean(profile) }),
    [allowedModules, hasPermission, profile, userAcl]
  );

  return <ACLContext.Provider value={value}>{children}</ACLContext.Provider>;
}
```

---

### 5️⃣ Aplicando o ACL no layout

Com o provider pronto, o layout protegido passou a consumir `allowedModules`.

Isso resolveu duas coisas:

- esconder itens de navegação sem permissão
- impedir acesso direto a uma URL não permitida

```tsx
const { allowedModules, isReady } = useACL();

const navItems = allowedModules.map((mod) => ({
  icon: mod.icon,
  label: mod.label,
  href: mod.href,
  active: mod.matchPath ? mod.matchPath(location) : location.startsWith(mod.href),
}));
```

E o redirecionamento ficou assim:

```tsx
useEffect(() => {
  if (!isReady) {
    return;
  }

  const currentModule = allowedModules.find((mod) =>
    mod.matchPath ? mod.matchPath(location) : location.startsWith(mod.href)
  );

  if (currentModule) {
    return;
  }

  const fallbackModule = allowedModules[0];
  if (!fallbackModule || fallbackModule.href === location) {
    return;
  }

  router.replace(fallbackModule.href);
}, [allowedModules, isReady, location, router]);
```

---

## ✅ O que essa implementação resolve

Depois dessa estrutura, o frontend passou a ter:

- menu lateral baseado em permissões reais
- navegação protegida por contexto
- fallback seguro enquanto o perfil ainda carrega
- validação de permissões desconhecidas
- uma API simples para a UI com `hasPermission()` e `allowedModules`

Além disso, qualquer componente pode usar a mesma base para renderização condicional.

---

## 🧩 Como usar dentro de componentes

Depois que o ACL está pronto, o consumo fica simples:

```tsx
import { useACL, PERMISSIONS } from "@/app/shared/acl";

export function ExampleButton() {
  const { hasPermission } = useACL();

  if (!hasPermission(PERMISSIONS.USERS)) {
    return null;
  }

  return <button>Criar usuário</button>;
}
```

Isso funciona muito bem para:

- botões de criar
- ações de editar
- tabs administrativas
- blocos sensíveis de UI

---

## 📁 O que você precisa para isso funcionar

Se quiser implementar esse modelo em outro projeto, os elementos mínimos são:

### Backend

- autenticação com token
- endpoint `/auth/me`
- retorno com `role` e `permissions`

### Frontend

- contexto de autenticação
- catálogo de permissões
- registro de módulos
- provider de ACL
- layout protegido usando `allowedModules`

Sem esses blocos, a solução fica incompleta.

---

## 🎯 Resultado

O ganho principal dessa abordagem foi sair de um ACL visual e mockado para um ACL realmente conectado ao estado autenticado do usuário.

Na prática:

- o frontend respeita a API
- o menu reflete o que o usuário pode acessar
- o acesso direto por URL deixa de ser “solto”
- a regra de autorização fica centralizada

E o melhor é que essa base cresce bem. Depois disso, fica fácil adicionar:

- `Can` component
- guardas por rota
- autorização por ação
- testes unitários do ACL

---

**Adendo :**

Uma coisa que achei interessante nessa implementação é que ela não tenta transformar o frontend em autoridade de segurança. O backend continua sendo quem decide de verdade. O papel do ACL aqui é organizar a experiência da interface, evitar navegação inconsistente e manter o produto coerente com o perfil autenticado. Para UI administrativa, CRM, ERP ou qualquer dashboard com múltiplos perfis, isso já resolve uma boa parte da dor. 

---

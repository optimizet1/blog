# Migrating from React to Solid.js: A Real-World Transformation

In today’s fast-paced digital world, building scalable, high-performance applications that are also easy to maintain is a major challenge. My previous team, with a frontend team of about 30 developers, nearly a year ago starter a major migration project: rewriting a large-scale, complex React application into **Solid.js**.

The original application had been developed over 4 years ago and since then multiple features have been added. It used a monolithic GraphQL BFF (Backend For Frontend) and served a wide range of features built over time by multiple teams. However, as the app grew, it became increasingly difficult to maintain. Adding new functionality was painful, performance was degrading, and user satisfaction scores were dropping due to slow data retrieval and unnecessary roundtrips.

I was one of four developers responsible for creating the initial version of this project and setting up the base platform that the rest of the team would later build upon. This meant defining our design system, setting up the routing and shell architecture, creating reusable utilities, and preparing the initial infrastructure that would power future development.

Having seen the original React application in its current complex state evolve and technical debt—I knew we had a unique opportunity to reset, and I took that responsibility seriously.

Let me walk you through the most important part: why I personally voted to use Solid.js.

What drew me in was not just its performance or its smaller size, but its philosophy: fewer abstractions, less magic, and a reactivity model that simply made more sense. I was tired of the hidden complexity of React's rendering lifecycle, of wrangling useEffect dependencies, and of spending time reasoning about why things rendered twice—or not at all. With Solid.js, those headaches simply went away. It gave us a framework that offered predictable updates, cleaner logic, and drastically lower cognitive load. That kind of developer experience is invaluable.

## 🚨 Why We Reconsidered React

Despite React’s success in the early years, several challenges emerged:
- Complex state management patterns (hooks, reducers, context layering).
- Tight coupling to a monolithic GraphQL BFF made UI logic harder to change without backend coordination.
- Inconsistent performance due to over-fetching and expensive client-side computations.
- Accumulated technical debt and bloated shared components.

After extensive evaluation—including trials of Svelte, Vue 3, and Solid.js—we found that **Solid.js offered the most compelling benefits**.

## ✅ Why Solid.js Won

- **Familiarity for React developers**: JSX, components, and one-way data flow.
- **No virtual DOM**: Updates go directly to the DOM, making the app noticeably faster.
- **Fine-grained reactivity**: `createSignal`, `createEffect`, and `createMemo` are easier to reason about than React hooks.
- **Reduced mental overhead**: Developers no longer had to juggle `useEffect` dependencies or worry about stale closures.
  -   In React, you need to manage state and side effects using a series of hooks like `useState`, `useEffect`, `useMemo`, and `useCallback`. These hooks often introduce complexity, especially in managing dependencies and avoiding unnecessary renders. In contrast, Solid uses **reactive primitives** such as `createSignal` and `createEffect` which behave more like observable values. 
  
    -  This means you don't need to worry about dependency arrays, stale closures, or re-running effects unnecessarily. State changes are automatically tracked, and updates propagate only where they need to go — making your code simpler, more predictable, and easier to reason about. 

   -  You get the same power of reactivity with far less boilerplate and mental overhead. In real-world scenarios, this can translate into 30-50% less code related to state management. Even for a small team of 4 developers working over 2-4 sprints on new features once the project is setup, Solid.js can reduce overall development time by up to 25%, minimizing coordination overhead and increasing feature velocity.
     
- **Simpler state management**: Eliminated the need for multiple state libraries and complex prop drilling.
- **Smaller bundle size**: Better initial download and load times.
- **Improved developer happiness**: Teams found Solid.js easier to debug, extend, and test.

## 🧱 A Multi-Phased, Modular Migration Plan

We didn’t rewrite everything in one go. Instead, we:
1. **Built foundational architecture**: Introduced a shell UI, routing system, layout framework, and navigation.
2. **Adopted a micro frontend philosophy using Solid.js** components as isolated islands.
3. **Created a modular service-based architecture**: Teams owned their own end-to-end stack: UI + Web Service (REST or GraphQL).
4. **Defined fracture planes**: Clear software boundaries inspired by the book *Team Topologies*, allowing each team to work independently.

## 🧠 Fracture Planes and Team Boundaries

With 5 cross-functional teams (4-5 developers each), we:
- Defined **bounded contexts** per feature area (e.g., dashboard, billing, reporting).
- Ensured teams owned the entire stack, including deployment of both UI and backend services.
- Used fracture planes to prevent accidental dependencies between teams.
- Agreed on **strict interface contracts** (API schemas, component props, shared types).

This allowed true autonomy and reduced coordination overhead. Developers could innovate and release independently.

## 🧩 Common Foundation and Integration Layer

Each team plugged into a shared foundational canvas that included:
- Main shell with menus and sub-navigation.
- Standard layout templates.
- Shared UI components (buttons, modals, alerts).
- Common utility libraries (date formatting, error handling, logging).

To avoid tight coupling, we established guidelines:

### 🔸 Use well-typed props and interfaces
```tsx
// components/UserCard.tsx
export interface UserCardProps {
  name: string;
  email: string;
  onSendMessage: (email: string) => void;
}

export function UserCard(props: UserCardProps) {
  return (
    <div class="user-card">
      <p>{props.name}</p>
      <p>{props.email}</p>
      <button onClick={() => props.onSendMessage(props.email)}>Message</button>
    </div>
  );
}
```

```tsx
// pages/ProfilePage.tsx
import { UserCard } from "../components/UserCard";

function ProfilePage() {
  const sendEmail = (email: string) => {
    console.log(`Sending message to ${email}`);
  };

  return <UserCard name="Alice Doe" email="alice@example.com" onSendMessage={sendEmail} />;
}
```

### 🔸 Use scoped context for broader state sharing
```tsx
// contexts/UserContext.tsx
import { createContext, useContext, ParentComponent, createSignal } from "solid-js";

type User = { name: string; email: string };
const UserContext = createContext<() => User>();

export const UserProvider: ParentComponent = (props) => {
  const [user] = createSignal<User>({ name: "Alice", email: "alice@example.com" });
  return <UserContext.Provider value={user}>{props.children}</UserContext.Provider>;
};

export function useUser() {
  const ctx = useContext(UserContext);
  if (!ctx) throw new Error("useUser must be used within a UserProvider");
  return ctx;
}
```

```tsx
// components/Greeting.tsx
import { useUser } from "../contexts/UserContext";

export function Greeting() {
  const user = useUser();
  return <h2>Welcome back, {user().name}!</h2>;
}
```

### 🔸 Use lightweight pub/sub for cross-boundary communication
```ts
// utils/eventBus.ts

type EventHandler<T = unknown> = (payload: T) => void;

class EventBus {
  private listeners = new Map<string, Set<EventHandler>>();

  subscribe<T>(event: string, handler: EventHandler<T>) {
    if (!this.listeners.has(event)) this.listeners.set(event, new Set());
    this.listeners.get(event)?.add(handler as EventHandler);
  }

  unsubscribe<T>(event: string, handler: EventHandler<T>) {
    this.listeners.get(event)?.delete(handler as EventHandler);
  }

  publish<T>(event: string, payload: T) {
    this.listeners.get(event)?.forEach((handler) => handler(payload));
  }
}

export const eventBus = new EventBus();
```

```tsx
// featureA/Notifier.tsx
import { eventBus } from "../utils/eventBus";

export function Notifier() {
  const handleClick = () => {
    eventBus.publish("USER_SAVED", { userId: 42 });
  };

  return <button onClick={handleClick}>Save User</button>;
}
```

```tsx
// featureB/Logger.tsx
import { onCleanup } from "solid-js";
import { eventBus } from "../utils/eventBus";

export function Logger() {
  const handler = (data: { userId: number }) => {
    console.log("User saved:", data.userId);
  };

  eventBus.subscribe("USER_SAVED", handler);
  onCleanup(() => eventBus.unsubscribe("USER_SAVED", handler));

  return null;
}
```

## 🔄 Data Communication Best Practices in Solid.js

Solid.js makes data flow simple and transparent:
- Components use `createSignal` for local reactive state.
- Shared data sources can use `createContext` + `useContext`, scoped to a feature or page.
- Computed values use `createMemo`, enabling highly optimized derived state.
- One team can expose reactive stores or signals that other teams can subscribe to, if necessary, while still respecting boundaries.

## 🔚 Results After Migration

- Reduced app startup time by ~40%.
- Developer onboarding and productivity increased by ~30%.
- Features delivered faster thanks to better architecture and lower cognitive load.
- State management issues and bugs dropped significantly.
- Easier to test, deploy, and scale independently.

## 🏁 Final Thoughts

Selecting Solid.js was a massive decision, but the benefits were clear. Solid.js would give our teams the performance we needed, drastically reduced complexity, and unlocked new levels of developer autonomy and velocity.

Shortly after laying the foundational groundwork—building the platform, defining architecture patterns, and equipping the team with clear migration paths—I transitioned to another critical initiative. The stage and plans were in place, and the teams had everything they needed to begin their journey with Solid.js.

I haven’t checked back in to see how the full implementation unfolded or whether the final app has fully realized all the promised perks of Solid.js. But what I can say with confidence is this: any new SPA that I architect or implement from scratch will have Solid.js as my first choice.

Of course, real-world project requirements may bring in other technologies and considerations, but when it comes down to React vs. Solid.js, my decision is already made. Less complexity, faster implementation, a cleaner reactive model—it’s a no-brainer. Solid.js is my top choice for any modern SPA that values maintainability, performance, and developer experience.

If you're facing increasing friction in your React codebase, consider piloting Solid.js in a new feature module. It could be the clean slate your team needs.

---
**Recommended Reading:**
- [Solid.js Official Docs](https://www.solidjs.com/docs)
- [Team Topologies by Matthew Skelton & Manuel Pais](https://teamtopologies.com)
- [Create Signals and Context in Solid](https://www.solidjs.com/docs/latest/api#createSignal)


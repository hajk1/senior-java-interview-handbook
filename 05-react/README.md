# React — Senior Interview Guide

This chapter covers the React questions expected from senior full-stack candidates in Java-backed teams. Strong answers explain what the rendering model and hooks actually do at runtime, not just the syntax—and connect that to performance, correctness, and testing consequences.

> **How to answer:** state the rule, explain the rendering or lifecycle mechanism behind it, then give a concrete bug or performance consequence it prevents or causes.

## Contents

1. [Components and rendering fundamentals](#1-components-and-rendering-fundamentals)
2. [State, props, and hooks](#2-state-props-and-hooks)
3. [Effects and lifecycle](#3-effects-and-lifecycle)
4. [Context, state management, and server state](#4-context-state-management-and-server-state)
5. [Performance](#5-performance)
6. [Forms, accessibility, and security](#6-forms-accessibility-and-security)
7. [Testing](#7-testing)
8. [Production and design scenarios](#8-production-and-design-scenarios)
9. [Rapid revision](#9-rapid-revision)

---

## 1. Components and rendering fundamentals

### 1. What is JSX, and what does it compile down to?

JSX is syntax sugar compiled (by Babel or the TypeScript compiler) into `React.createElement(type, props, ...children)` calls—or, with the modern JSX runtime, into calls to `jsx`/`jsxs` imported automatically from `react/jsx-runtime`. Either way, JSX is not HTML at runtime; it produces plain JavaScript objects describing what should be rendered.

### 2. What is the virtual DOM, and how does reconciliation use it?

The virtual DOM is a lightweight tree of React elements describing the desired UI. On each render, React builds a new element tree and diffs it against the previous one (reconciliation) to compute the minimal set of real DOM mutations needed, rather than re-creating the actual DOM from scratch.

The diff is heuristic, not a general tree-diff algorithm: React assumes elements of a different type produce different subtrees (and remounts rather than diffs them), and it relies on the `key` prop to match items within a list across renders.

### 3. How does the `key` prop affect reconciliation, and what breaks with array-index keys?

`key` tells React which element in a new list corresponds to which element in the old list, independent of position. Without a stable key, or with the array index as the key, inserting or removing an item in the middle of a list shifts every subsequent item's identity, so React reuses the wrong DOM node and component state for the shifted position—visible as state (like an open accordion or an input's contents) “sticking” to the wrong row after a reorder.

Use a stable identifier from the data (an id) as the key, not the array index, whenever the list can be reordered, filtered, or have items inserted/removed.

### 4. What is component identity, and why does changing a component's type or position remount it?

React associates state with a position in the element tree combined with the component type at that position, not with the component function itself. If the type at a given position changes between renders (conditionally rendering `<Modal>` versus `<Drawer>` at the same spot, or moving a component to a different parent), React unmounts the old instance—discarding its state and effects—and mounts a new one, even if the two components look similar.

This is why conditionally swapping between visually similar components can unexpectedly reset local state (a text input losing its value) where the developer expected the same component to just update.

### 5. What actually triggers a component to re-render?

A component re-renders when its own state changes, when its parent re-renders (by default, regardless of whether the props actually changed), or when it consumes a context whose value changed. Props changing is not itself the trigger—a parent re-rendering with identical props still re-renders the child unless the child is memoized.

### 6. Why are React elements immutable, and what does that make safe?

An element returned by a render is a plain, frozen-in-practice description of what should be rendered—React never mutates it after creation. This is what makes it safe to compare (`Object.is`) as part of memoization strategies (`React.memo`, `useMemo`) and to keep around previous-render snapshots for diffing without them changing out from under the diff.

### 7. State versus props—what is the actual distinction?

Props are read-only input supplied by a parent; a component cannot modify its own props. State is data owned and mutated by the component itself (or lifted to and owned by an ancestor) that persists across renders and, when changed, schedules a re-render. Treating props as mutable, or storing a prop's value in state and never syncing it back, is a common source of “my UI didn't update” bugs.

### 8. Why did hooks replace class lifecycle methods?

Class lifecycle methods (`componentDidMount`, `componentDidUpdate`, `componentWillUnmount`) group code by *when* it runs rather than by *what concern* it belongs to, so one concern (a subscription's setup and teardown) ends up split across two or three methods, while one method often mixes several unrelated concerns. Hooks let you colocate a concern's full lifecycle—setup, update handling, and cleanup—in one `useEffect`, and let logic be extracted and reused as a plain function (a custom hook) without the inheritance and `this`-binding baggage classes required.

---

## 2. State, props, and hooks

### 9. How does `useState` preserve state across renders, and what does a “stale closure” mean?

React stores each hook's state in a slot on the component's internal fiber, keyed by call order, not by variable name—which is why hooks must be called unconditionally in the same order every render. Each render's function body is a fresh closure over that render's props and state; a callback created in one render (e.g., inside a `setTimeout` or an event handler) that captures a state variable will keep referencing that render's value even after state has since changed, unless it reads state via the functional updater or a ref.

### 10. Why can't hooks be called conditionally?

React matches hook calls to their stored state by the order they're called in, not by name—so if a hook is skipped on some renders (inside an `if`, after an early `return`, or in a loop), every subsequent hook call in that render shifts to the wrong stored slot. This corrupts state and effects silently in some cases and throws an “order of hooks” error in others; the rule exists specifically to keep call order stable across renders.

### 11. What is a custom hook, and what does it actually share—state or logic?

A custom hook is a plain function that calls other hooks and follows the `useXxx` naming convention. It shares *logic and pattern*, not state: each component that calls a custom hook gets its own independent instance of whatever state or effects that hook sets up internally, just as calling `useState` twice in two different components gives two independent pieces of state.

### 12. `useState` versus `useReducer`?

`useState` fits independent, simple pieces of state updated directly. `useReducer` centralizes update logic into one pure reducer function, which helps when several state transitions are related, when the next state depends on multiple pieces of the previous state together, or when you want the update logic itself to be easily testable in isolation from rendering.

### 13. What is the functional update form of `setState`, and when is it required?

`setCount(c => c + 1)` computes the next state from the latest state at the time the update is applied, rather than from whatever value was captured in the closure when the updater was created. It is required whenever a state update depends on the previous state and might run after other pending updates—multiple rapid calls in the same event handler, or an update scheduled from a stale closure (a timer, a websocket callback)—because `setCount(count + 1)` would reuse the same captured `count` for every call.

### 14. How does React batch state updates, and what changed in React 18?

Batching groups multiple `setState` calls into a single re-render instead of one render per call. Before React 18, batching only happened inside React event handlers; state updates inside a `setTimeout`, a promise callback, or a raw DOM event listener each triggered their own synchronous render. React 18's automatic batching extends this to those cases as well, so updates from async code are batched too unless explicitly opted out with `flushSync`.

### 15. What is prop drilling, and when is it acceptable versus a smell?

Prop drilling is passing a value through several intermediate components that don't use it themselves, purely to reach a descendant that does. It is acceptable for a shallow tree or when the intermediate components are meant to be generic and reusable regardless of that prop. It becomes a smell when it forces many unrelated components to know about data that isn't conceptually theirs, or when the chain is deep enough that a rename touches every intermediate component—at that point, composition (passing the already-rendered child as a prop) or context is usually a better fit.

### 16. What is `useRef` for, and why doesn't updating it cause a re-render?

`useRef` returns a mutable object (`{ current: ... }`) that persists across renders without being part of the render output—useful for holding a DOM node reference, a mutable value that shouldn't trigger re-rendering (a timer id, a previous-value cache), or an “instance variable” equivalent from class components. Mutating `.current` doesn't schedule a re-render because refs are explicitly outside React's state/render-tracking mechanism; that is the entire point of using one instead of `useState`.

---

## 3. Effects and lifecycle

### 17. What problem does `useEffect` solve, and how does it differ from the class lifecycle methods it replaced?

`useEffect` synchronizes a component with an external system (a subscription, a fetch, a DOM API, a timer) after the render is committed to the screen. Unlike `componentDidMount`/`componentDidUpdate`, which are two separate methods for two separate moments, one `useEffect` call handles the initial synchronization and every subsequent re-synchronization through the same function body—React re-runs the whole effect (after cleaning up the previous one) rather than asking you to separately compute “what changed.”

### 18. What is the dependency array, and what bugs come from omitting a dependency?

The dependency array tells React when to re-run the effect: React compares each dependency to its previous-render value with `Object.is` and re-runs the effect if any changed. Omitting a value the effect actually reads causes the effect to keep using a stale closure over that value's first-render version—a classic symptom is an interval or event handler that only ever sees the initial state, silently ignoring updates.

```jsx
useEffect(() => {
  const id = setInterval(() => console.log(count), 1000); // stale `count` if omitted
  return () => clearInterval(id);
}, [count]);
```

### 19. What does the cleanup function do, and when does it run?

A function returned from the effect runs before the effect re-runs (to undo whatever the previous run set up) and again on unmount. It exists because an effect that subscribes, opens a connection, or sets a timer would otherwise leak or duplicate that resource every time the effect re-runs—cleanup is how React keeps “one active subscription” true across repeated synchronizations, not just at unmount.

### 20. Why does React invoke effects twice in development (under `StrictMode`), and what does that expose?

`StrictMode` intentionally mounts, unmounts, and remounts each component once extra in development, running effect setup, cleanup, and setup again, to surface effects whose cleanup is missing or incorrect—an effect that only ever ran once in practice can hide a resource leak that becomes visible once React proves the setup/cleanup pair is repeatable. It is a development-only diagnostic; production builds do not double-invoke effects.

### 21. How do you avoid a race condition in a data-fetching effect?

Guard against a stale response overwriting a newer one—either with an `ignore`/`cancelled` flag set in cleanup and checked before applying the result, or with an `AbortController` aborted in cleanup:

```jsx
useEffect(() => {
  let ignore = false;
  fetchUser(id).then(data => { if (!ignore) setUser(data); });
  return () => { ignore = true; };
}, [id]);
```

Without this, rapidly changing `id` (fast navigation, fast typing into a search box) can let an earlier, slower request resolve after a later, faster one, leaving stale data displayed as if it were current.

### 22. `useEffect` versus `useLayoutEffect`?

`useEffect` runs asynchronously after the browser has painted the updated DOM, so it doesn't block visual updates. `useLayoutEffect` runs synchronously after DOM mutations but before the browser paints, which is necessary when the effect needs to measure or mutate the DOM (layout, scroll position, avoiding a visible flicker) before the user sees an intermediate state. Reaching for `useLayoutEffect` by default costs paint-blocking time that most effects don't need.

### 23. Why do some effect dependencies want to be “read but not reactive”?

Some values an effect reads (like a logging function, or an id used only inside an event handler defined in the effect) shouldn't cause the effect to re-run when they change—they're needed for the effect's *behavior* but aren't part of what it's *synchronizing with*. Wrapping such reads in an “effect event” (a stable-identity function that always sees the latest values without being a reactive dependency) avoids the false choice between an incomplete dependency array (stale closures) and an overly broad one (unnecessary re-runs).

---

## 4. Context, state management, and server state

### 24. What problem does Context solve, and what is its main performance pitfall?

Context lets a value be read by any descendant without passing it through every intermediate component's props, useful for cross-cutting concerns like theme, locale, or the current authenticated user. Its main pitfall is that every consumer of a context re-renders whenever the provider's value changes—even if the consumer only cares about part of it—so a context holding a large or frequently changing object turns any single field's update into a re-render storm across every consumer.

### 25. When should you reach for a global state library instead of Context?

Context is a dependency-injection mechanism, not a state-management library—it has no built-in support for selective subscriptions, middleware, or update batching beyond what plain React gives you. Reach for a dedicated library when many unrelated components need to read *different slices* of shared state and re-render independently (which requires selector-based subscriptions Context doesn't provide out of the box), or when you need time-travel debugging, persistence, or complex cross-cutting update logic.

### 26. What is “server state,” and why do libraries like React Query treat it differently from client state?

Server state is data your application doesn't own—it lives on a server, can change without your code changing it, and is fetched asynchronously with its own loading/error/staleness lifecycle. Treating it as ordinary `useState`/Context client state means hand-rolling caching, deduplication of concurrent requests, background refetching, and staleness—which is exactly what a server-state library provides so component code can describe *what* data it needs rather than managing the fetch lifecycle itself.

### 27. What does the `useReducer` + Context pattern buy you over passing individual setters down?

It gives descendants a single stable `dispatch` function (whose identity never changes) instead of multiple setter functions that may change identity across renders, and it centralizes “what state transitions are valid” in one reducer rather than scattering direct state mutations across every consumer. Providing `dispatch` via context (separately from the state value itself) also means components that only dispatch actions, and never read state, don't re-render when that state changes.

### 28. What is state colocation, and why does lifting state up too eagerly hurt performance?

Colocation means keeping state as close as possible to the components that actually use it, lifting it to a shared ancestor only when multiple components genuinely need to read or coordinate on it. Lifting state higher than necessary means every update re-renders that ancestor and, by default, every descendant beneath it—turning a change that should affect one small subtree into a re-render of a much larger one.

### 29. How do you avoid unnecessary context re-renders?

Split one large context into several narrower ones so a component only subscribes to (and re-renders for) the slice it actually reads, and memoize the value object passed to the provider (`useMemo`) so an unrelated parent re-render doesn't hand consumers a brand-new object reference every time even when the underlying values haven't changed.

### 30. Optimistic updates—what can go wrong?

An optimistic update shows the expected result immediately, before the server confirms it, to make the UI feel instant. What can go wrong: the server rejects or returns a different result than assumed (requiring a correct rollback path, not just “hope it succeeds”), concurrent optimistic updates can race and reconcile in the wrong order, and an optimistic update to a list (adding/removing an item) needs a stable temporary identity so it can be matched up with or replaced by the real server response.

---

## 5. Performance

### 31. `React.memo`—what does it actually prevent, and when is it useless?

`React.memo` skips re-rendering a component if its props are shallow-equal to the previous render's props, which prevents the *child's own render function* from re-running when its parent re-renders with unchanged props. It is useless (or actively counterproductive, given the comparison cost) if the component almost always receives new prop references anyway—commonly because the parent passes an inline object, array, or function literal that is a new reference on every render regardless of whether the underlying data changed.

### 32. `useMemo` versus `useCallback`—what do they actually memoize?

`useMemo` memoizes a computed *value*, recomputing it only when its dependencies change—useful for expensive calculations or to keep a stable object/array reference. `useCallback` memoizes a *function reference* itself (it is essentially `useMemo` specialized for functions), which matters when that function is passed to a memoized child or into another hook's dependency array, where a new reference on every render would otherwise defeat the memoization or re-trigger an effect.

### 33. Why can passing a new inline function or object as a prop defeat `memo`?

`React.memo`'s default comparison is shallow (`Object.is` per prop). An inline arrow function or object literal (`onClick={() => doThing()}`, `style={{ color: 'red' }}`) creates a new reference every render, so even though the memoized child's *other* props are unchanged, this one prop always compares as different—causing the “memoized” child to re-render on every parent render anyway. Stabilizing that prop's identity with `useCallback`/`useMemo` (or moving the literal outside the render, if it doesn't depend on render-scoped values) is what actually makes the memoization effective.

### 34. What is code splitting, and how do `React.lazy` and `Suspense` implement it?

Code splitting breaks the JavaScript bundle into smaller chunks loaded on demand instead of all upfront, so the initial page load only pays for the code it actually needs. `React.lazy(() => import('./Component'))` defines a component whose module is fetched lazily on first render; wrapping it in `<Suspense fallback={...}>` lets React show a fallback UI while that chunk is still loading, rather than the component needing to manage its own loading state for the import itself.

### 35. What is list virtualization, and when is it necessary?

Virtualization renders only the list items currently visible in the viewport (plus a small buffer), recycling DOM nodes as the user scrolls, instead of mounting every item in a long list at once. It becomes necessary once a list is long enough (hundreds to thousands of rows, depending on row complexity) that mounting every item creates a slow initial render and a large, sluggish DOM—symptoms that don't show up with a 20-item list in development but appear immediately with production-scale data.

### 36. How do you profile a React app to find unnecessary re-renders?

Use the React DevTools Profiler to record a render pass and inspect which components rendered, how long each took, and (with “why did this render” style highlighting) what changed to trigger it. Look specifically for components re-rendering with props that are logically unchanged (usually a broken memoization due to unstable references) versus components that are genuinely expensive per render and are candidates for `useMemo` inside them or virtualization.

### 37. What is the actual performance cost being paid—reconciliation, or the render function running?

Both are real but distinct costs: running a component's render function (executing its JavaScript, including any expensive computation inside it) happens for every component whose render wasn't skipped, and reconciliation (diffing the resulting element tree, and committing the minimal DOM mutations) happens afterward. Memoization (`memo`, `useMemo`) targets the first cost by skipping unnecessary render function calls or recomputation; the diff/commit cost is usually smaller unless the tree is very large or deeply nested, which is where virtualization and reducing tree size help more than memoization does.

### 38. Concurrent rendering—what do `startTransition` and `useDeferredValue` actually change?

They let you mark some state updates as lower priority, so React can interrupt rendering that update to handle a more urgent one (like the next keystroke) instead of finishing a slow render first and making input feel laggy. `startTransition` marks the *update* as non-urgent (React may render the urgent update first and come back to the transition); `useDeferredValue` gives you a value that lags behind the latest one during busy renders, useful when you don't control where the state update itself happens but do control how a slow-to-render value derived from it is consumed.

---

## 6. Forms, accessibility, and security

### 39. Controlled versus uncontrolled components?

A controlled component's value is driven entirely by React state (`value` + `onChange`), so React is the single source of truth and every keystroke round-trips through a re-render. An uncontrolled component lets the DOM hold its own value, read on demand via a ref (or on submit) instead of on every change.

Controlled inputs make validation, formatting, and conditional logic straightforward at the cost of a re-render per keystroke; uncontrolled inputs avoid that cost and suit simple forms or integrating non-React widgets, at the cost of losing per-keystroke reactivity.

### 40. How should form validation and error state be structured for good UX?

Distinguish “this field is currently invalid” from “show the user an error”—validating on every keystroke but only displaying the error after a field is blurred or the form is submitted avoids showing an error before the user has finished typing. Validation state should be colocated with the field it validates where possible, with cross-field or async validation (uniqueness checks, server-side rules) handled explicitly and debounced rather than firing on every keystroke.

### 41. What accessibility basics does a custom form component need to preserve?

Every input needs a programmatically associated label (`<label htmlFor>` or `aria-label`), keyboard operability equivalent to a native element (focus order, `Enter`/`Space` activation, arrow-key navigation for composite widgets), visible focus indication, and error messages associated with their field (`aria-describedby`) and announced to assistive technology (`aria-live` or `role="alert"`) rather than conveyed by color or icon alone. Building a custom dropdown or date picker from `div`s means re-implementing all of this by hand—native elements provide it for free.

### 42. What is a security risk specific to React, and how does it differ from templating-based frameworks?

`dangerouslySetInnerHTML` (and, to a lesser extent, setting `href`/`src` from unsanitized user input) bypasses React's default escaping and injects raw HTML/JS, creating an XSS vector if the content isn't sanitized. This is opt-in and explicitly named to be hard to reach for by accident—unlike some templating systems that escape by default only for certain contexts and can silently pass raw HTML through others, React's default (interpolating `{value}` as text) is safe, and the risk is concentrated in the few explicitly-named unsafe escape hatches.

### 43. How does React handle focus management, and what commonly breaks it?

React doesn't manage focus itself—focus is a browser/DOM concept, and a focused element keeps focus across a re-render only if the same DOM node survives reconciliation. Focus commonly breaks when a re-render causes the focused element to remount (a changed `key`, a changed component type at that position, or replacing a node with an equivalent-looking new one) since a new DOM node cannot inherit focus from the one it replaced—the fix is preserving the element's identity across the render, or explicitly re-focusing via a ref in an effect.

### 44. What is the risk of storing sensitive tokens in state or `localStorage` versus cookies?

Anything readable by JavaScript—state, `localStorage`, `sessionStorage`—is also readable by any script that manages to run on the page via XSS, making a stored auth token directly exfiltratable. An `HttpOnly` cookie is not readable by JavaScript at all, which removes that specific exposure (though cookies bring their own concerns—CSRF—that need mitigation, typically `SameSite` plus a CSRF token). The general principle: don't hold a credential in JS-accessible storage if a mechanism exists to keep it out of JS's reach entirely.

### 45. How do you handle async validation without race conditions?

Debounce the trigger (don't fire a request per keystroke) and guard the response the same way as any other async effect—track which request is the latest (a request id, or an abort controller) and discard a response that isn't from the most recent request. Without this, a slower earlier validation call can resolve after a faster later one, flipping a field's error state back to a stale result for input the user has already changed.

---

## 7. Testing

### 46. What does “test like a user” (React Testing Library's philosophy) mean in practice?

Query and interact with the rendered output the way a user would—by visible text, label, or role—rather than reaching into component internals (state, props, instance methods, CSS class names used purely for styling). This makes tests resilient to internal refactors (renaming a prop, changing from `useState` to `useReducer`) as long as the user-facing behavior is unchanged, and it fails a test exactly when a real user would notice something broken.

### 47. Shallow versus full rendering—why is shallow rendering usually discouraged now?

Shallow rendering renders only one component level, stubbing out children, which lets a test isolate a component completely from its children's implementation—but it also means the test can't verify what the user actually sees or how the component integrates with real children, and it couples tightly to the tree structure rather than behavior. Full rendering (mounting the real component tree, as Testing Library encourages) verifies actual rendered output and interaction, which is closer to what matters and more resilient to internal refactors.

### 48. How do you test a component with async data fetching?

Render the component, then use an async query (`findBy...`, or `waitFor`) that retries until the expected UI appears, rather than asserting immediately after render—the fetch hasn't resolved yet at that point. Mock the network layer (a mock server intercepting requests, or a mocked fetch/client) rather than mocking the component's internal fetching hook, so the test still exercises the real request/response handling code.

### 49. Unit, integration, and end-to-end tests for a frontend app—what does each actually catch?

A unit test isolates one function or component (a reducer, a pure formatting utility) and is fast and precise about what broke, but says nothing about integration. An integration test renders a meaningful slice of the UI together (a form plus its validation plus its submission handler) and catches wiring bugs a unit test can't see. An end-to-end test drives the real running application through a real browser against a real (or realistically mocked) backend, catching issues no lower-level test can—actual build output, real network behavior, cross-browser quirks—at the cost of being slower and more expensive to maintain.

### 50. How do you avoid brittle tests coupled to implementation details?

Assert on user-observable outcomes (what's rendered, what happens after an interaction) instead of internal state, prop values, or call counts on internal functions; query the DOM the way a user would identify elements (role, label, visible text) rather than by CSS class or test-only internal structure that has no bearing on behavior. A test that breaks because a component was refactored from hooks to a different internal structure, with no change in behavior, was coupled to the wrong layer.

### 51. How do you mock network requests in component tests?

Intercept at the network boundary (a request-mocking layer that intercepts actual `fetch`/XHR calls and returns configured responses) rather than mocking your own data-fetching hook or client function directly. Intercepting at the network boundary still exercises your real request-building, response-parsing, and error-handling code, whereas mocking your own fetching function only tests that the component called it—which passes even if that function is subtly broken.

### 52. What is a snapshot test good for, and what is it bad for?

It's good for catching *unintentional* changes to a stable, simple output (a small presentational component, a serialized data structure)—a diff shows exactly what changed for a human to judge as intended or not. It's bad as a substitute for asserting actual behavior: a large snapshot is reviewed by nobody in practice (developers reflexively approve/update it), and it doesn't verify the output is *correct*, only that it's the same as last time—including if it was already wrong.

---

## 8. Production and design scenarios

### 53. A list re-renders entirely on every keystroke in an unrelated input. How do you diagnose and fix it?

Check whether the input's state lives at a common ancestor of both the input and the list—if so, every keystroke re-renders that ancestor and, by default, everything beneath it, including the unrelated list. Diagnose with the React DevTools Profiler to confirm the list is actually re-rendering and why; fix by colocating the input's state closer to just the input (so the list's ancestor doesn't re-render at all), or by memoizing the list component if it must stay under the same state.

### 54. An effect causes an infinite re-render loop. What are the likely causes?

Most commonly: the effect sets state that is also in its own dependency array without a condition that eventually stops updating it, or the dependency array contains a new object/array/function reference created fresh every render (so it never “stabilizes” and the effect always sees a changed dependency). Fix by conditioning the state update, moving the unstable value out of the dependency (or memoizing it), or reconsidering whether the value needs to be a dependency at all versus read via a ref.

### 55. A component's data is stale after navigating back to it. What's happening?

If the component was unmounted while navigating away and remounted on return, any effect-driven fetch should have re-run on mount—so stale data on return usually means the data source is cached indefinitely without invalidation (a naive in-memory cache with no staleness policy), or the component wasn't actually remounted and its effect's dependencies didn't change to trigger a refetch. Server-state libraries address exactly this with configurable staleness and refetch-on-focus/mount policies instead of ad hoc caching.

### 56. How would you design a large form with many interdependent fields for performance and correctness?

Avoid one giant state object updated on every keystroke re-rendering the entire form—colocate state per field or per logical section where fields don't need to see each other's latest value on every render, and centralize only the genuinely cross-field logic (validation rules that span fields, computed totals) in one place, ideally with a reducer so the transition logic is testable independent of rendering. Debounce expensive derived computations and any async validation, and make sure error display timing (immediate versus on-blur/submit) is consistent across the form rather than field-by-field accidental behavior.

### 57. How do you decide whether a piece of state belongs local, lifted, or in a global store/server-state cache?

Start local; lift it only as far as the nearest common ancestor of the components that actually need to read or coordinate on it—not further “just in case.” Reach for a global client-state store only when state is genuinely needed across distant, unrelated parts of the tree with no natural common ancestor worth re-rendering. Anything that represents data owned by a server, rather than transient UI state, belongs in a server-state cache regardless of how many components read it, because its lifecycle (fetching, staleness, invalidation) is fundamentally different from client state.

### 58. What distinguishes a senior React answer?

- Explains *why* a re-render happens (identity, position, context) rather than only how to prevent it.
- Treats effects as synchronization with an external system, not a lifecycle-method replacement.
- Distinguishes client state, derived state, and server state, and doesn't default everything into one bucket.
- Knows what memoization actually compares and when it's a net cost instead of a win.
- Builds forms and custom widgets to preserve the accessibility a native element would have given for free.
- Tests observable behavior, not internal implementation details.
- Diagnoses performance and correctness bugs with the Profiler and reasoning about renders, not guesswork.

---

## 9. Rapid revision

### Must-answer questions

Before an interview, answer these without notes:

1. What does JSX compile to?
2. How does reconciliation use the `key` prop?
3. Why does changing a component's type or position remount it?
4. What actually triggers a re-render?
5. Why do hooks need a stable call order every render?
6. What is a stale closure, and when does the functional `setState` form fix it?
7. `useState` versus `useReducer`?
8. What changed with automatic batching in React 18?
9. What does an effect's cleanup function actually undo?
10. Why does `StrictMode` double-invoke effects in development?
11. How do you avoid a race condition in a data-fetching effect?
12. `useEffect` versus `useLayoutEffect`?
13. What is Context's main performance pitfall?
14. When do you need a global state library instead of Context?
15. Why is server state handled differently from client state?
16. How do you reduce unnecessary context re-renders?
17. What does `React.memo` actually compare?
18. `useMemo` versus `useCallback`?
19. Why can an inline prop defeat memoization?
20. What is list virtualization, and when do you need it?
21. What do `startTransition`/`useDeferredValue` actually change?
22. Controlled versus uncontrolled components?
23. What accessibility basics does a custom form widget need?
24. What is React's main XSS risk, and how is it different from templating frameworks?
25. Why do stored tokens in JS-accessible storage carry more XSS risk than `HttpOnly` cookies?
26. What does “test like a user” mean?
27. What does each test level (unit/integration/e2e) actually catch?
28. Where should you mock in a component test with network calls?
29. What causes an infinite effect re-render loop?
30. How do you decide where a piece of state should live?

### Thirty-second summary

React re-renders based on state, parent re-renders, and context changes, then reconciles the new element tree against the old one using type and `key` to decide what to reuse versus remount—component identity, not just visual similarity, is what state survives across. Hooks colocate a concern's full lifecycle in one function, but depend on a stable call order and correct dependency tracking to avoid stale closures, race conditions, and infinite loops. Performance work is mostly about avoiding unnecessary render function calls (memoization with stable references) and reducing tree size (virtualization, code splitting) rather than the diff itself. Client state, derived state, and server state have different correct homes and lifecycles, and senior-level React answers connect all of this to accessibility, security, and tests that verify user-observable behavior rather than implementation details.

## Official references

- [React documentation](https://react.dev/)
- [React reference: rendering and commit phases](https://react.dev/learn/render-and-commit)
- [React reference: Effects](https://react.dev/learn/synchronizing-with-effects)
- [Testing Library guiding principles](https://testing-library.com/docs/guiding-principles/)
- [Web Content Accessibility Guidelines (WCAG)](https://www.w3.org/WAI/standards-guidelines/wcag/)
- [OWASP XSS prevention cheat sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)

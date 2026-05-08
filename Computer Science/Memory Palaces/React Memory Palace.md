
This is the routing note for React memory-palace learning. The individual React concept notes should hold the detailed explanation, imagery, examples, and flashcards. This file should stay as the palace map: rooms, topic groupings, and retrieval order.

## Room 1: Entrance And Setup

Purpose: orient yourself before building. This room answers what React is, why it exists, and how a project begins.

Suggested `palace-room`: `Entrance And Setup`

| Order | Topic                                            | Existing note?      |
| ----- | ------------------------------------------------ | ------------------- |
| 100   | [[React - What is React (The Why)]]              | yes                 |
| 110   | [[React - Setting Up a Development Environment]] | yes                 |
| 120   | [[React - Using Create React App (CRA)]]         | yes                 |
| 130   | [[React - Vite]]                                 | yes                 |

Suggested future loci:

- Front door: what React is and why it exists.
- Shoe rack: choosing a project setup.
- Old toolbox: Create React App.
- Fast workbench: Vite.

```dataview
TABLE locus as "Locus", palace-order as "Order"
FROM "Computer Science/React"
WHERE palace = "Finowstraße" AND palace-room = "Entrance"
SORT palace-order ASC
```

## Room 2: Component Workbench

Purpose: encode the component-first way of thinking. This room should make React feel like assembling UI from reusable parts.

Suggested `palace-room`: `Component Workbench`

| Order | Topic | Existing note? |
|---|---|---|
| 200 | [[React - Thinking in Components]] | yes |
| 210 | [[React - Components]] | roadmap link |
| 220 | [[React - Functional Components]] | roadmap link |
| 230 | [[React - Class Components (Legacy)]] | roadmap link |
| 240 | [[React - JSX (JavaScript XML)]] | yes |
| 250 | [[React - Embedding Expressions in JSX]] | roadmap link |
| 260 | [[React - JSX Attributes]] | roadmap link |
| 270 | [[React - Rendering Elements]] | roadmap link |

Suggested future loci:

- Workbench surface: thinking in components.
- Parts bins: components.
- Modern hand tool: functional components.
- Old machine: class components.
- Blueprint sheet: JSX.
- Inked labels: JSX attributes.
- Display stand: rendered elements.

```dataview
TABLE locus as "Locus", palace-order as "Order"
FROM "Computer Science/React"
WHERE palace = "React Memory Palace" AND palace-room = "Component Workbench"
SORT palace-order ASC
```

## Room 3: Props, State, And Rendering

Purpose: encode the core data-flow model: parent data, internal data, UI updates, and list rendering.

Suggested `palace-room`: `Props State And Rendering`

| Order | Topic | Existing note? |
|---|---|---|
| 300 | [[React - Props (Properties)]] | yes |
| 310 | [[React - Passing and Accessing Props]] | roadmap link |
| 320 | [[React - Prop Types (for validation)]] | roadmap link |
| 330 | [[React - State]] | yes |
| 340 | [[React - The `useState` Hook]] | yes |
| 350 | [[React - `setState` in Class Components (Legacy)]] | roadmap link |
| 360 | [[React - Understanding Immutability]] | roadmap link |
| 370 | [[React - Conditional Rendering]] | roadmap link |
| 380 | [[React - `if-else` Statements]] | roadmap link |
| 390 | [[React - Ternary Operators]] | roadmap link |
| 400 | [[React - Logical `&&` Operator]] | roadmap link |
| 410 | [[React - Lists and Keys]] | roadmap link |
| 420 | [[React - The `map()` method for rendering]] | roadmap link |
| 430 | [[React - The Importance of `key` Props]] | roadmap link |
| 440 | [[React - The Virtual DOM (VDOM)]] | yes |

Suggested future loci:

- Mail chute from parent: props.
- Inbox label: accessing props.
- Security stamp: prop validation.
- Thermostat: state.
- State dial: `useState`.
- Frozen glass case: immutability.
- Forked hallway: conditional rendering.
- Numbered coat hooks: lists and keys.
- Shadow screen: Virtual DOM.

```dataview
TABLE locus as "Locus", palace-order as "Order"
FROM "Computer Science/React"
WHERE palace = "React Memory Palace" AND palace-room = "Props State And Rendering"
SORT palace-order ASC
```

## Room 4: Hooks Pier

Purpose: group hooks as the functional-component control system. This can reuse the fishing pier imagery already present in `React - Introduction to Hooks`.

Suggested `palace-room`: `Hooks Pier`

| Order | Topic | Existing note? |
|---|---|---|
| 500 | [[React - Introduction to Hooks]] | yes |
| 510 | [[React - The `useState` Hook (Revisited)]] | roadmap link |
| 520 | [[React - The `useEffect` Hook]] | roadmap link |
| 530 | [[React - `useEffect` with Dependencies]] | roadmap link |
| 540 | [[React - `useEffect` for Cleanup]] | roadmap link |
| 550 | [[React - The `useContext` Hook]] | roadmap link |
| 560 | [[React - The `useReducer` Hook]] | roadmap link |
| 570 | [[React - The `useRef` Hook]] | roadmap link |
| 580 | [[React - The `useMemo` and `useCallback` Hooks]] | roadmap link |
| 590 | [[React - Custom Hooks]] | roadmap link |

Suggested future loci:

- Pier entrance sign: rules of hooks.
- Tackle box: `useState`.
- Ripples in water: `useEffect`.
- Tide chart: dependency array.
- Net cleanup station: cleanup functions.
- Shared bait bucket: `useContext`.
- Gear mechanism: `useReducer`.
- Fishing pole handle: `useRef`.
- Preserved bait freezer: `useMemo` and `useCallback`.
- Handmade lure: custom hooks.

```dataview
TABLE locus as "Locus", palace-order as "Order"
FROM "Computer Science/React"
WHERE palace = "React Memory Palace" AND palace-room = "Hooks Pier"
SORT palace-order ASC
```

## Room 5: Events And Forms Counter

Purpose: encode user interaction: clicks, input, forms, controlled data, uncontrolled data, and shared state.

Suggested `palace-room`: `Events And Forms Counter`

| Order | Topic | Existing note? |
|---|---|---|
| 600 | [[React - Handling Events]] | roadmap link |
| 610 | [[React - Forms]] | roadmap link |
| 620 | [[React - Controlled Components]] | roadmap link |
| 630 | [[React - Uncontrolled Components]] | roadmap link |
| 640 | [[React - Lifting State Up]] | roadmap link |

Suggested future loci:

- Service bell: events.
- Paper form stack: forms.
- Clerk holding the pen: controlled components.
- Customer holding the pen: uncontrolled components.
- Pulley lifting a box upward: lifting state up.

```dataview
TABLE locus as "Locus", palace-order as "Order"
FROM "Computer Science/React"
WHERE palace = "React Memory Palace" AND palace-room = "Events And Forms Counter"
SORT palace-order ASC
```

## Room 6: Router Station

Purpose: encode SPA navigation and React Router. This room should feel like moving through paths without a full page reload.

Suggested `palace-room`: `Router Station`

| Order | Topic | Existing note? |
|---|---|---|
| 700 | [[React - What is a Single Page Application (SPA)]] | yes |
| 710 | [[React - Setting up React Router]] | roadmap link |
| 720 | [[React - Core Components]] | roadmap link |
| 730 | [[React - `<BrowserRouter>`]] | roadmap link |
| 740 | [[React - `<Routes>` Component]] | roadmap link |
| 750 | [[React - `<Route>` Component]] | roadmap link |
| 760 | [[React - `path` prop]] | roadmap link |
| 770 | [[React - `element` prop]] | roadmap link |
| 780 | [[React - Navigation]] | roadmap link |
| 790 | [[React - `<Link>` Component]] | roadmap link |
| 800 | [[React - NavLink Component]] | yes |
| 810 | [[React - Advanced Routing]] | roadmap link |
| 820 | [[React - Nested Routes]] | roadmap link |
| 830 | [[React - Dynamic Routes (URL Parameters)]] | roadmap link |
| 840 | [[React - Programmatic Navigation]] | roadmap link |
| 850 | [[React - Handling "Not Found" (404) Pages]] | roadmap link |

Suggested future loci:

- Station map: SPA.
- Ticket booth: router setup.
- Main terminal: `BrowserRouter`.
- Route board: `Routes`.
- Platform gate: `Route`.
- Track number: `path`.
- Train car: `element`.
- Walking bridge: `Link`.
- Glowing active sign: `NavLink`.
- Branching stairwell: nested routes.
- Variable platform sign: dynamic routes.
- Control lever: programmatic navigation.
- Lost-and-found desk: 404 page.

```dataview
TABLE locus as "Locus", palace-order as "Order"
FROM "Computer Science/React"
WHERE palace = "React Memory Palace" AND palace-room = "Router Station"
SORT palace-order ASC
```

## Room 7: Data Fetching Control Room

Purpose: encode client-server data flow, loading states, errors, and when state needs a dedicated library.

Suggested `palace-room`: `Data Fetching Control Room`

| Order | Topic | Existing note? |
|---|---|---|
| 900 | [[React - Data Fetching with the Fetch API]] | roadmap link |
| 910 | [[React - Using `useEffect` for API Calls]] | roadmap link |
| 920 | [[React - Handling Loading and Error States]] | roadmap link |
| 930 | [[React - Using `Axios`]] | roadmap link |
| 940 | [[React - Axios]] | yes |
| 950 | [[React - Advanced State Management]] | roadmap link |
| 960 | [[React - When do you need a state management library?]] | roadmap link |
| 970 | [[React - Redux Toolkit (RTK)]] | roadmap link |
| 980 | [[React - Zustand]] | roadmap link |
| 990 | [[React - React Query (TanStack Query)]] | roadmap link |

Suggested future loci:

- Network cable: Fetch API.
- Scheduled monitor: `useEffect` for API calls.
- Spinner light: loading state.
- Alarm panel: error state.
- Request console: Axios.
- Global control board: advanced state management.
- Heavy machinery switch: Redux Toolkit.
- Small lightweight switch: Zustand.
- Cache cabinet: React Query.

```dataview
TABLE locus as "Locus", palace-order as "Order"
FROM "Computer Science/React"
WHERE palace = "React Memory Palace" AND palace-room = "Data Fetching Control Room"
SORT palace-order ASC
```

## Room 8: Styling Studio

Purpose: encode the different ways React components receive styling.

Suggested `palace-room`: `Styling Studio`

| Order | Topic | Existing note? |
|---|---|---|
| 1000 | [[React - Inline Styles]] | roadmap link |
| 1010 | [[React - CSS Modules]] | roadmap link |
| 1020 | [[React - CSS-in-JS Libraries]] | roadmap link |
| 1030 | [[React - Styled-Components]] | roadmap link |
| 1040 | [[React - Emotion]] | roadmap link |
| 1050 | [[React - Utility-First CSS (Tailwind CSS)]] | roadmap link |

Suggested future loci:

- Paintbrush in hand: inline styles.
- Labeled paint cabinet: CSS Modules.
- Paint mixer machine: CSS-in-JS.
- Designer mannequin: Styled Components.
- Mood color wall: Emotion.
- Utility pegboard: Tailwind CSS.

```dataview
TABLE locus as "Locus", palace-order as "Order"
FROM "Computer Science/React"
WHERE palace = "React Memory Palace" AND palace-room = "Styling Studio"
SORT palace-order ASC
```

## Room 9: Testing Lab

Purpose: encode testing from the user's perspective: what exists, what behavior matters, and how interaction is simulated.

Suggested `palace-room`: `Testing Lab`

| Order | Topic | Existing note? |
|---|---|---|
| 1100 | [[React - Introduction to Testing in React]] | roadmap link |
| 1110 | [[React - Jest]] | roadmap link |
| 1120 | [[React - React Testing Library]] | roadmap link |
| 1130 | [[React - Writing Unit Tests for Components]] | roadmap link |
| 1140 | [[React - Simulating Events and Interactions]] | roadmap link |

Suggested future loci:

- Lab entrance checklist: intro to testing.
- Test runner bench: Jest.
- User observation window: React Testing Library.
- Microscope: unit tests.
- Robot hand: simulated events.

```dataview
TABLE locus as "Locus", palace-order as "Order"
FROM "Computer Science/React"
WHERE palace = "React Memory Palace" AND palace-room = "Testing Lab"
SORT palace-order ASC
```

## Room 10: Performance Garage

Purpose: encode optimization tools and when they matter.

Suggested `palace-room`: `Performance Garage`

| Order | Topic | Existing note? |
|---|---|---|
| 1200 | [[React - `React.memo` for component memoization]] | roadmap link |
| 1210 | [[React - `useMemo` and `useCallback` (Revisited)]] | roadmap link |
| 1220 | [[React - Code Splitting with `React.lazy` and `Suspense`]] | roadmap link |
| 1230 | [[React - Profiling with the React Developer Tools]] | roadmap link |

Suggested future loci:

- Parked car with cover: `React.memo`.
- Saved engine parts: `useMemo` and `useCallback`.
- Split garage door: code splitting.
- Diagnostic scanner: React DevTools profiler.

```dataview
TABLE locus as "Locus", palace-order as "Order"
FROM "Computer Science/React"
WHERE palace = "React Memory Palace" AND palace-room = "Performance Garage"
SORT palace-order ASC
```

## Room 11: Advanced Patterns Balcony

Purpose: encode advanced or legacy patterns as things you recognize and can compare, even if you do not use them every day.

Suggested `palace-room`: `Advanced Patterns Balcony`

| Order | Topic | Existing note? |
|---|---|---|
| 1300 | [[React - Higher-Order Components (HOCs) - Legacy Pattern]] | roadmap link |
| 1310 | [[React - Render Props - Legacy Pattern]] | roadmap link |
| 1320 | [[React - Context API (Deep Dive)]] | roadmap link |
| 1330 | [[React - Portals]] | roadmap link |
| 1340 | [[React - Error Boundaries]] | roadmap link |
| 1350 | [[React - Forwarding Refs]] | roadmap link |

Suggested future loci:

- Wrapper curtain: HOCs.
- Gift pass-through window: render props.
- Broadcast speaker: Context API.
- Balcony trapdoor: portals.
- Safety net: error boundaries.
- Extended pointer rod: forwarding refs.

```dataview
TABLE locus as "Locus", palace-order as "Order"
FROM "Computer Science/React"
WHERE palace = "React Memory Palace" AND palace-room = "Advanced Patterns Balcony"
SORT palace-order ASC
```

## Room 12: Project Yard

Purpose: attach abstract React concepts to real projects. Use this room after individual concepts are encoded.

Suggested `palace-room`: `Project Yard`

| Order | Topic | Existing note? |
|---|---|---|
| 1400 | [[Project: Interactive To-Do List]] | roadmap link |
| 1410 | [[Project: Blog with React Router and API Data]] | roadmap link |
| 1420 | [[Project: E-commerce Shopping Cart]] | roadmap link |
| 1430 | [[Project: Portfolio Website]] | roadmap link |

Suggested future loci:

- Clipboard: to-do list.
- Newspaper stand: blog with router and API data.
- Shopping basket: e-commerce cart.
- Display wall: portfolio website.

```dataview
TABLE locus as "Locus", palace-order as "Order"
FROM "Computer Science/React"
WHERE palace = "React Memory Palace" AND palace-room = "Project Yard"
SORT palace-order ASC
```

## Existing React Notes To Assign First

These are the current physical files in the React folder. Assign these before creating new notes, because they give the palace useful traction immediately.

| Suggested order | Note | Suggested room |
|---|---|---|
| 100 | [[React - What is React (The Why)]] | Entrance And Setup |
| 110 | [[React - Setting Up a Development Environment]] | Entrance And Setup |
| 120 | [[React - Using Create React App (CRA)]] | Entrance And Setup |
| 130 | [[React - Vite]] | Entrance And Setup |
| 200 | [[React - Thinking in Components]] | Component Workbench |
| 240 | [[React - JSX (JavaScript XML)]] | Component Workbench |
| 300 | [[React - Props (Properties)]] | Props State And Rendering |
| 330 | [[React - State]] | Props State And Rendering |
| 340 | [[React - The `useState` Hook]] | Props State And Rendering |
| 440 | [[React - The Virtual DOM (VDOM)]] | Props State And Rendering |
| 500 | [[React - Introduction to Hooks]] | Hooks Pier |
| 700 | [[React - What is a Single Page Application (SPA)]] | Router Station |
| 800 | [[React - NavLink Component]] | Router Station |
| 940 | [[React - Axios]] | Data Fetching Control Room |

## Assignment Checklist

- [ ] Add `palace: React Memory Palace` to each existing React note.
- [ ] Add one of the `palace-room` values from this file.
- [ ] Choose a concrete `locus` for each note.
- [ ] Add a numeric `palace-order` using the suggested order values.
- [ ] Keep each note's `# Memory Palace` section to 1-3 loci.
- [ ] Add at least one flashcard that directly tests the palace retrieval path.

Example palace-retrieval flashcard:

What is the retrieval path for React state in the React Memory Palace?;;Enter Props State And Rendering -> go to the thermostat -> recall that state is internal component data that changes over time and triggers re-rendering

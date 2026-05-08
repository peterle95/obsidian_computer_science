---
memory: to_finish
tags: 
language: 
review-date: 
last-reviewed: 
keywords: 
scheda: to_finish
---
# React Learning Roadmap

## 1. 🚀 Core Concepts & Fundamentals

- **[[React - What is React (The Why)]]**: Understanding the purpose and benefits of using React.
- **[[React - Thinking in Components]]**: The core philosophy of building UIs with React.
- **[[React - Setting Up a Development Environment]]**
    - **[[React - Using Create React App (CRA)]]**: The traditional, feature-rich setup.
    - **[[React - Vite]]**: A modern, faster alternative for project setup.
- **[[React - JSX (JavaScript XML)]]**: The syntax for writing HTML within JavaScript.
    - [[React - Embedding Expressions in JSX]]
    - [[React - JSX Attributes]]
    - [[React - Rendering Elements]]
- **[[React - Components]]**: The building blocks of a React application.
    - **[[React - Functional Components]]**: Modern components using functions.
    - **[[React - Class Components (Legacy)]]**: Understanding older, class-based syntax.
- **[[React - Props (Properties)]]**: Passing data from parent to child components.
    - [[React - Passing and Accessing Props]]
    - [[React - Prop Types (for validation)]]
- **[[React - State]]**: Managing a component's internal data.
    - **[[React - The `useState` Hook]]**: The primary way to manage state in functional components.
    - [[React - `setState` in Class Components (Legacy)]]
    - [[React - Understanding Immutability]]
- **[[React - Conditional Rendering]]**: Displaying different UI based on conditions.
    - [[React - `if-else` Statements]]
    - [[React - Ternary Operators]]
    - [[React - Logical `&&` Operator]]
- **[[React - Lists and Keys]]**: Rendering dynamic lists of elements.
    - [[React - The `map()` method for rendering]]
    - [[React - The Importance of `key` Props]]

---

## 2. 🎣 React Hooks (The "How" for Functional Components)

- **[[React - Introduction to Hooks]]**: Why hooks were introduced.
- **[[React - The `useState` Hook (Revisited)]]**: Deep dive into state management.
- **[[React - The `useEffect` Hook]]**: Handling side effects (data fetching, subscriptions, etc.).
    - [[React - `useEffect` with Dependencies]]
    - [[React - `useEffect` for Cleanup]]
- **[[React - The `useContext` Hook]]**: Managing global state without "prop drilling".
- **[[React - The `useReducer` Hook]]**: An alternative to `useState` for complex state logic.
- **[[React - The `useRef` Hook]]**: Accessing DOM nodes or persisting values across renders.
- **[[React - The `useMemo` and `useCallback` Hooks]]**: Optimizing performance by memoizing values and functions.
- **[[React - Custom Hooks]]**: Creating your own reusable hooks.

---

## 3. 🌐 Handling Events & Forms

- **[[React - Handling Events]]**: Managing user interactions like clicks and input changes.
- **[[React - Forms]]**: Working with form elements.
    - **[[React - Controlled Components]]**: Forms where React state controls the input values.
    - **[[React - Uncontrolled Components]]**: Forms where the DOM handles the data (often using `useRef`).
- **[[React - Lifting State Up]]**: Sharing state between sibling components.

---

## 4. 🧭 Routing with React Router

- **[[React - What is a Single Page Application (SPA)?]]**
- **[[React - Setting up React Router]]**
- **[[React - Core Components]]**
    - **[[React - `<BrowserRouter>`]]**
    - **[[React - `<Routes>` Component]]**: A container for all your routes.
    - **[[React - `<Route>` Component]]**: Mapping a path to a component.
        - [[React - `path` prop]]
        - [[React - `element` prop]]
- **[[React - Navigation]]**
    - **[[React - `<Link>` Component]]**: Basic navigation.
    - **[[React - NavLink Component]]**: A link that knows if it's "active".
- **[[React - Advanced Routing]]**
    - **[[React - Nested Routes]]**: Structuring routes within each other.
    - **[[React - Dynamic Routes (URL Parameters)]]**: Creating routes like `/users/:userId`.
    - **[[React - Programmatic Navigation]]**: Navigating using code (the `useNavigate` hook).
    - **[[React - Handling "Not Found" (404) Pages]]**

---

## 5. 🔄 Data Fetching & State Management

- **[[React - Data Fetching with the Fetch API]]**
- **[[React - Using `useEffect` for API Calls]]**
- **[[React - Handling Loading and Error States]]**
- **[[React - Using `Axios`]]**: A popular library for making HTTP requests.
- **[[React - Advanced State Management]]**
    - **[[React - When do you need a state management library?]]**
    - **[[React - Redux Toolkit (RTK)]]**: The standard for complex state management.
    - **[[React - Zustand]]**: A simpler, more modern alternative to Redux.
    - **[[React - React Query (TanStack Query)]]**: For managing server state (fetching, caching, synchronizing).

---

## 6. 🛠️ Styling in React

- **[[React - Inline Styles]]**
- **[[React - CSS Modules]]**: Scoping CSS locally to components.
- **[[React - CSS-in-JS Libraries]]**
    - [[React - Styled-Components]]
    - [[React - Emotion]]
- **[[React - Utility-First CSS (Tailwind CSS)]]**: A popular framework for rapid styling.

---

## 7. 🧪 Testing Your Application

- **[[React - Introduction to Testing in React]]**
- **[[React - Jest]]**: The underlying testing framework.
- **[[React - React Testing Library]]**: For testing components from a user's perspective.
- **[[React - Writing Unit Tests for Components]]**
- **[[React - Simulating Events and Interactions]]**

---

## 8. 🚀 Performance Optimization

- **[[React - `React.memo` for component memoization]]**
- **[[React - `useMemo` and `useCallback` (Revisited)]]**
- **[[React - Code Splitting with `React.lazy` and `Suspense`]]**: Loading components only when needed.
- **[[React - Profiling with the React Developer Tools]]**

---

## 9. 🏛️ Advanced Concepts & Patterns

- **[[React - Higher-Order Components (HOCs) - Legacy Pattern]]**
- **[[React - Render Props - Legacy Pattern]]**
- **[[React - Context API (Deep Dive)]]**: Understanding the provider/consumer model.
- **[[React - Portals]]**: Rendering components outside their parent DOM hierarchy.
- **[[React - Error Boundaries]]**: Gracefully handling errors in components.
- **[[React - Forwarding Refs]]**: Passing refs through components to child elements.

---

## 🌱 My React Projects

- **[[Project: Interactive To-Do List]]**
- **[[Project: Blog with React Router and API Data]]**
- **[[Project: E-commerce Shopping Cart]]**
- **[[Project: Portfolio Website]]**

## 🚀 Beyond React: Production Frameworks
- **[[Next.js - Full-Stack React Framework]]**: Server-side rendering, routing, and full-stack capabilities
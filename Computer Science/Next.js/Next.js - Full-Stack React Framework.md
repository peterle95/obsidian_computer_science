---
memory: to_finish
tags: 
language: 
review-date: 
last-reviewed: ""
scheda: to_finish
visit-count: 0
confidence-level: 1
consecutive-correct: 0
last-struggle-date: ""
cssclasses:
---

## Prerequisites
- Solid understanding of React fundamentals (components, hooks, state management)
- Basic knowledge of JavaScript ES6+ features
- Familiarity with Node.js and npm/yarn
- Understanding of HTML, CSS, and basic web concepts

---

## 1. 🚀 Next.js Fundamentals

### **[[Next.js - What is Next.js and Why Use It?]]**
- Understanding the difference between React (library) vs Next.js (framework)
- Key benefits: SSR, SSG, performance optimizations, developer experience
- When to choose Next.js over Create React App

### **[[Next.js - Setting Up Your Development Environment]]**
- Installing Next.js with `create-next-app`
- Project structure overview
- Development server and hot reloading
- **[[Next.js - App Router vs Pages Router]]**: Understanding the two routing paradigms

### **[[Next.js - File-Based Routing System]]**
- How file structure determines routes
- **[[Next.js - Pages and Layouts]]**
- **[[Next.js - Dynamic Routes]]**: `[id].js`, `[...slug].js`
- **[[Next.js - Route Groups]]**: `(folder)` organization
- **[[Next.js - Nested Routes]]**: Folder-based nesting

---

## 2. 🎨 Rendering Strategies

### **[[Next.js - Server-Side Rendering (SSR)]]**
- Understanding when and why to use SSR
- **[[Next.js - `getServerSideProps`]]** (Pages Router)
- Performance implications and use cases

### **[[Next.js - Static Site Generation (SSG)]]**
- **[[Next.js - `getStaticProps`]]** (Pages Router)
- **[[Next.js - `getStaticPaths`]]** for dynamic static pages
- **[[Next.js - Incremental Static Regeneration (ISR)]]**

### **[[Next.js - Client-Side Rendering (CSR)]]**
- When to use traditional React rendering
- **[[Next.js - `useEffect` for Client-Side Data Fetching]]**

### **[[Next.js - App Router Rendering]]** (Next.js 13+)
- **[[Next.js - Server Components vs Client Components]]**
- **[[Next.js - `async/await` in Server Components]]**
- **[[Next.js - `use client` directive]]**

---

## 3. 🧭 Advanced Routing & Navigation

### **[[Next.js - Navigation Components]]**
- **[[Next.js - `<Link>` Component]]**: Client-side navigation
- **[[Next.js - `useRouter` Hook]]**: Programmatic navigation
- **[[Next.js - Router Events]]**: Handling route changes

### **[[Next.js - Advanced Routing Patterns]]**
- **[[Next.js - Catch-All Routes]]**: `[...slug].js`
- **[[Next.js - Optional Catch-All Routes]]**: `[[...slug]].js`
- **[[Next.js - Route Handlers]]** (App Router): API-like functionality
- **[[Next.js - Middleware]]**: Request/response interception

---

## 4. 🎯 Data Fetching & Management

### **[[Next.js - Data Fetching Strategies]]**
- Choosing between SSR, SSG, and CSR for your use case
- **[[Next.js - `fetch` API with Next.js]]**
- **[[Next.js - Data Caching and Revalidation]]**

### **[[Next.js - API Routes]]**
- **[[Next.js - Creating API Endpoints]]**: `/api` folder structure
- **[[Next.js - Handling HTTP Methods]]**: GET, POST, PUT, DELETE
- **[[Next.js - API Route Handlers]]** (App Router)
- **[[Next.js - Database Integration]]**: Connecting to databases

### **[[Next.js - External Data Integration]]**
- **[[Next.js - Working with Headless CMS]]**
- **[[Next.js - Third-Party API Integration]]**
- **[[Next.js - Authentication]]**: NextAuth.js, custom auth solutions

---

## 5. 🎨 Styling & Assets

### **[[Next.js - CSS and Styling Options]]**
- **[[Next.js - CSS Modules]]**: Component-scoped styling
- **[[Next.js - Global CSS]]**: Application-wide styles
- **[[Next.js - CSS-in-JS]]**: Styled-components, Emotion integration
- **[[Next.js - Tailwind CSS Integration]]**

### **[[Next.js - Image and Asset Optimization]]**
- **[[Next.js - `<Image>` Component]]**: Automatic optimization
- **[[Next.js - Static Assets]]**: `/public` folder usage
- **[[Next.js - Font Optimization]]**: Google Fonts and custom fonts

---

## 6. ⚡ Performance & Optimization

### **[[Next.js - Built-in Performance Features]]**
- **[[Next.js - Automatic Code Splitting]]**
- **[[Next.js - Bundle Analyzer]]**: Analyzing bundle size
- **[[Next.js - Core Web Vitals]]**: Measuring performance

### **[[Next.js - Advanced Optimization]]**
- **[[Next.js - Dynamic Imports]]**: `next/dynamic`
- **[[Next.js - Script Optimization]]**: `<Script>` component
- **[[Next.js - Caching Strategies]]**: Browser and CDN caching

---

## 7. 🚀 Deployment & Production

### **[[Next.js - Deployment Options]]**
- **[[Next.js - Vercel Deployment]]**: The native platform
- **[[Next.js - Self-Hosting]]**: Node.js servers, Docker
- **[[Next.js - Static Export]]**: `next export` for static hosting

### **[[Next.js - Environment Configuration]]**
- **[[Next.js - Environment Variables]]**: `.env.local`, public vs private
- **[[Next.js - Configuration File]]**: `next.config.js` customization

---

## 8. 🧪 Testing & Debugging

### **[[Next.js - Testing Strategies]]**
- **[[Next.js - Unit Testing]]**: Jest and React Testing Library
- **[[Next.js - Integration Testing]]**: Testing API routes
- **[[Next.js - E2E Testing]]**: Cypress, Playwright integration

### **[[Next.js - Debugging and Development Tools]]**
- **[[Next.js - Developer Tools]]**: React DevTools, Next.js DevTools
- **[[Next.js - Error Handling]]**: Error pages, error boundaries
- **[[Next.js - Logging and Monitoring]]**: Production debugging

---

## 9. 🏗️ Advanced Patterns & Architecture

### **[[Next.js - Application Architecture]]**
- **[[Next.js - Folder Structure Best Practices]]**
- **[[Next.js - Component Organization]]**
- **[[Next.js - State Management]]**: Redux, Zustand, React Query integration

### **[[Next.js - Advanced Features]]**
- **[[Next.js - Internationalization (i18n)]]**
- **[[Next.js - Progressive Web Apps (PWA)]]**
- **[[Next.js - Custom App and Document]]**: `_app.js`, `_document.js`

---

## 🌱 Next.js Projects

### **Beginner Projects**
- **[[Project: Personal Blog with SSG]]**
- **[[Project: Company Landing Page]]**

### **Intermediate Projects**
- **[[Project: E-commerce Store with API Routes]]**
- **[[Project: Dashboard with Authentication]]**

### **Advanced Projects**
- **[[Project: Full-Stack SaaS Application]]**
- **[[Project: Multi-tenant Platform]]**


### **Key Differences to Remember**
- File-based routing vs React Router
- Server vs client component considerations
- Data fetching patterns unique to Next.js
- Build-time vs runtime considerations
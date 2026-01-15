---
title: Use JSX for HTML Templating
impact: MEDIUM
impactDescription: faster and more maintainable than string concatenation
tags: jsx, templating, hono, html, performance
---

## Use JSX for HTML Templating

Hono's JSX support is faster than string concatenation and provides better type safety and code organization.

**Incorrect (string concatenation):**

```typescript
import { Hono } from 'hono'

const app = new Hono()

app.get('/', (c) => {
  const title = 'My App'
  const items = ['Item 1', 'Item 2', 'Item 3']

  // Error-prone and hard to maintain
  let html = '<html><head><title>' + title + '</title></head><body>'
  html += '<h1>' + title + '</h1><ul>'

  for (const item of items) {
    html += '<li>' + item + '</li>'
  }

  html += '</ul></body></html>'

  return c.html(html)
})
```

**Correct (using JSX):**

```typescript
import { Hono } from 'hono'
import { html } from 'hono/html'

const app = new Hono()

// Define reusable components
const Layout = (props: { children: any, title: string }) => {
  return (
    <html>
      <head>
        <title>{props.title}</title>
      </head>
      <body>{props.children}</body>
    </html>
  )
}

const ItemList = (props: { items: string[] }) => {
  return (
    <ul>
      {props.items.map(item => <li>{item}</li>)}
    </ul>
  )
}

app.get('/', (c) => {
  const title = 'My App'
  const items = ['Item 1', 'Item 2', 'Item 3']

  return c.html(
    <Layout title={title}>
      <h1>{title}</h1>
      <ItemList items={items} />
    </Layout>
  )
})
```

**Setup TypeScript for JSX:**

```typescript
// tsconfig.json
{
  "compilerOptions": {
    "jsx": "react-jsx",
    "jsxImportSource": "hono/jsx"
  }
}
```

**JSX with fragments:**

```typescript
const Header = () => {
  return (
    <>
      <meta charset="UTF-8" />
      <meta name="viewport" content="width=device-width, initial-scale=1.0" />
      <link rel="stylesheet" href="/style.css" />
    </>
  )
}

const Page = () => {
  return (
    <html>
      <head>
        <Header />
      </head>
      <body>
        <h1>Hello</h1>
      </body>
    </html>
  )
}
```

**Conditional rendering:**

```typescript
const UserProfile = (props: { user?: User }) => {
  return (
    <div>
      {props.user ? (
        <div>
          <h2>Welcome, {props.user.name}</h2>
          <p>Email: {props.user.email}</p>
        </div>
      ) : (
        <div>
          <h2>Please log in</h2>
          <a href="/login">Login</a>
        </div>
      )}
    </div>
  )
}
```

**Raw HTML (when needed):**

```typescript
import { raw } from 'hono/html'

const Page = (props: { content: string }) => {
  return (
    <div>
      {/* Escaped by default */}
      <p>{props.content}</p>

      {/* Raw HTML (use carefully!) */}
      <div>{raw(props.content)}</div>
    </div>
  )
}
```

**Async components (streaming):**

```typescript
const AsyncData = async () => {
  const data = await fetchData()

  return (
    <div>
      <h2>Data</h2>
      <pre>{JSON.stringify(data, null, 2)}</pre>
    </div>
  )
}

app.get('/data', async (c) => {
  return c.html(
    <Layout>
      <AsyncData />
    </Layout>
  )
})
```

**Type-safe props:**

```typescript
interface CardProps {
  title: string
  description: string
  imageUrl?: string
}

const Card = ({ title, description, imageUrl }: CardProps) => {
  return (
    <div class="card">
      {imageUrl && <img src={imageUrl} alt={title} />}
      <h3>{title}</h3>
      <p>{description}</p>
    </div>
  )
}

// TypeScript will error if props are missing or wrong type
app.get('/card', (c) => {
  return c.html(
    <Card
      title="My Card"
      description="Description here"
      imageUrl="/image.jpg"
    />
  )
})
```

**Using jsxRenderer middleware:**

```typescript
import { jsxRenderer } from 'hono/jsx-renderer'

app.use('*', jsxRenderer(({ children }) => {
  return (
    <html>
      <head>
        <meta charset="UTF-8" />
        <title>My App</title>
      </head>
      <body>{children}</body>
    </html>
  )
}))

// Routes automatically wrapped in layout
app.get('/', (c) => {
  return c.render(
    <div>
      <h1>Home Page</h1>
      <p>Content here</p>
    </div>
  )
})

app.get('/about', (c) => {
  return c.render(
    <div>
      <h1>About Page</h1>
      <p>About content</p>
    </div>
  )
})
```

**Performance considerations:**

```typescript
// Bad - creates new component each render
app.get('/', (c) => {
  const Item = (props: { text: string }) => <li>{props.text}</li>
  return c.html(<ul><Item text="Test" /></ul>)
})

// Good - define components once
const Item = (props: { text: string }) => <li>{props.text}</li>

app.get('/', (c) => {
  return c.html(<ul><Item text="Test" /></ul>)
})
```

Benefits:
- Type-safe props with TypeScript
- Better code organization
- Faster than string concatenation
- Automatic HTML escaping (XSS protection)
- Reusable components
- Better editor support

Reference: [Hono JSX](https://hono.dev/docs/guides/jsx)

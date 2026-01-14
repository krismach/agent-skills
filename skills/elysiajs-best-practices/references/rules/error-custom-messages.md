---
title: Provide Custom Error Messages in Schemas
impact: MEDIUM
impactDescription: Better user experience and debugging
tags: error, validation, schema, ux
---

## Provide Custom Error Messages in Schemas

**Impact: MEDIUM (Better user experience and debugging)**

Use the `error` property in TypeBox schemas to provide clear, user-friendly error messages instead of generic validation failures.

**Incorrect (generic error messages):**

```typescript
app.post('/user', ({ body }) => createUser(body), {
  body: t.Object({
    email: t.String({ format: 'email' }),
    age: t.Number({ minimum: 18 }),
    password: t.String({ minLength: 8 })
  })
})
// Returns: "Expected string to match 'email' format"
```

**Correct (custom error messages):**

```typescript
app.post('/user', ({ body }) => createUser(body), {
  body: t.Object({
    email: t.String({
      format: 'email',
      error: 'Please provide a valid email address'
    }),
    age: t.Number({
      minimum: 18,
      error: 'You must be at least 18 years old'
    }),
    password: t.String({
      minLength: 8,
      error: 'Password must be at least 8 characters long'
    })
  })
})
```

**Pattern: Contextual error messages:**

```typescript
app.post('/transfer', ({ body }) => processTransfer(body), {
  body: t.Object({
    amount: t.Number({
      minimum: 0.01,
      maximum: 10000,
      error: 'Transfer amount must be between $0.01 and $10,000'
    }),
    fromAccount: t.String({
      pattern: '^[0-9]{10}$',
      error: 'From account must be a 10-digit account number'
    }),
    toAccount: t.String({
      pattern: '^[0-9]{10}$',
      error: 'To account must be a 10-digit account number'
    })
  })
})
```

**Combined with error handler:**

```typescript
app
  .onError(({ code, error, set }) => {
    if (code === 'VALIDATION') {
      set.status = 400

      // Custom error messages are in error.all
      return {
        error: 'Validation failed',
        fields: error.all.map(e => ({
          field: e.path.replace('/body/', ''),
          message: e.schema.error || e.message
        }))
      }
    }
  })

  .post('/signup', ({ body }) => createAccount(body), {
    body: t.Object({
      username: t.String({
        minLength: 3,
        maxLength: 20,
        pattern: '^[a-zA-Z0-9_]+$',
        error: 'Username must be 3-20 characters (letters, numbers, underscore only)'
      }),
      email: t.String({
        format: 'email',
        error: 'Invalid email format'
      }),
      password: t.String({
        minLength: 12,
        pattern: '^(?=.*[a-z])(?=.*[A-Z])(?=.*[0-9])(?=.*[!@#$%^&*])',
        error: 'Password must be 12+ chars with uppercase, lowercase, number, and special character'
      })
    })
  })
```

**Localized error messages:**

```typescript
const errorMessages = {
  en: {
    emailInvalid: 'Please provide a valid email address',
    ageMinimum: 'You must be at least 18 years old',
    passwordLength: 'Password must be at least 8 characters'
  },
  es: {
    emailInvalid: 'Por favor proporcione una dirección de correo válida',
    ageMinimum: 'Debes tener al menos 18 años',
    passwordLength: 'La contraseña debe tener al menos 8 caracteres'
  }
}

app
  .derive(({ headers }) => ({
    lang: headers['accept-language']?.startsWith('es') ? 'es' : 'en'
  }))

  .post('/user', ({ body }) => createUser(body), ({ lang }) => ({
    body: t.Object({
      email: t.String({
        format: 'email',
        error: errorMessages[lang].emailInvalid
      }),
      age: t.Number({
        minimum: 18,
        error: errorMessages[lang].ageMinimum
      }),
      password: t.String({
        minLength: 8,
        error: errorMessages[lang].passwordLength
      })
    })
  }))
```

**Complex validation with helper function:**

```typescript
const passwordSchema = t.String({
  minLength: 12,
  maxLength: 128,
  pattern: '^(?=.*[a-z])(?=.*[A-Z])(?=.*[0-9])(?=.*[!@#$%^&*])',
  error: [
    'Password requirements:',
    '- At least 12 characters',
    '- One uppercase letter',
    '- One lowercase letter',
    '- One number',
    '- One special character (!@#$%^&*)'
  ].join('\n')
})

app.post('/change-password', ({ body }) => changePassword(body), {
  body: t.Object({
    currentPassword: t.String({ error: 'Current password is required' }),
    newPassword: passwordSchema
  })
})
```

Reference: [ElysiaJS Validation](https://elysiajs.com/essential/validation#custom-error)

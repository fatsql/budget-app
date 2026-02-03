# Budget App Architecture Guide - Complete System Design

## 🏗️ Architecture Overview

This document provides a complete architectural overview of the Budget App, covering frontend, backend, database, infrastructure, and deployment.

---

## 📐 High-Level Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                         USER DEVICES                                │
│  🖥️ Desktop    📱 Mobile    💻 Tablet                              │
└─────────────────────────┬──────────────────────────────────────────┘
                          │
                          │ HTTPS (SSL/TLS)
                          ▼
┌────────────────────────────────────────────────────────────────────┐
│                    CLOUD LOAD BALANCER                              │
│                     (Google Cloud CDN)                              │
│  • SSL Termination  • DDoS Protection  • Global Distribution       │
└─────────────────────────┬──────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────────────┐
│                      CLOUD RUN (Serverless)                         │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │              NEXT.JS APPLICATION                              │  │
│  │  ┌─────────────────┐    ┌──────────────────┐                │  │
│  │  │    FRONTEND     │    │     BACKEND      │                │  │
│  │  │   React Pages   │◄───┤   API Routes     │                │  │
│  │  │   Components    │    │   Business Logic │                │  │
│  │  │   State Mgmt    │    │   Auth Handler   │                │  │
│  │  └─────────────────┘    └──────────────────┘                │  │
│  └─────────────────────────────┬──────────────────────────────────┘  │
└────────────────────────────────┼──────────────────────────────────────┘
                                 │
                                 │ Private Network (VPC)
                                 │
           ┌─────────────────────┴──────────────────────┐
           │                                             │
           ▼                                             ▼
┌──────────────────────┐                    ┌──────────────────────┐
│   CLOUD SQL          │                    │   SECRET MANAGER     │
│   (PostgreSQL)       │                    │                      │
│  • Users             │                    │  • DB Credentials    │
│  • Transactions      │                    │  • API Keys          │
│  • Categories        │                    │  • Auth Secrets      │
│  • Budgets           │                    └──────────────────────┘
│  • Goals             │
└──────────────────────┘
```

---

## 🎨 Frontend Architecture (React/Next.js)

### Technology Stack

```yaml
Core Framework: Next.js 14 (App Router)
UI Library: React 18
Language: TypeScript
Styling: Tailwind CSS
State Management: React Context + Zustand
Forms: React Hook Form + Zod
Charts: Recharts / Chart.js
Authentication: NextAuth.js
HTTP Client: Fetch API / Axios
```

### Directory Structure

```
src/
├── app/                          # Next.js App Router
│   ├── (auth)/                   # Auth routes group
│   │   ├── login/
│   │   │   └── page.tsx          # Login page
│   │   └── register/
│   │       └── page.tsx          # Registration page
│   │
│   ├── (dashboard)/              # Protected routes group
│   │   ├── layout.tsx            # Dashboard layout with sidebar
│   │   ├── page.tsx              # Dashboard home
│   │   ├── transactions/
│   │   │   ├── page.tsx          # Transactions list
│   │   │   └── [id]/
│   │   │       └── page.tsx      # Transaction detail
│   │   ├── budgets/
│   │   │   └── page.tsx          # Budget management
│   │   ├── goals/
│   │   │   └── page.tsx          # Savings goals
│   │   └── settings/
│   │       └── page.tsx          # User settings
│   │
│   ├── api/                      # API routes (Backend)
│   │   ├── auth/
│   │   │   └── [...nextauth]/
│   │   │       └── route.ts      # NextAuth handler
│   │   ├── transactions/
│   │   │   ├── route.ts          # GET all, POST new
│   │   │   └── [id]/
│   │   │       └── route.ts      # GET, PUT, DELETE by ID
│   │   ├── budgets/
│   │   │   └── route.ts
│   │   ├── goals/
│   │   │   └── route.ts
│   │   └── dashboard/
│   │       └── route.ts          # Dashboard summary
│   │
│   ├── layout.tsx                # Root layout
│   ├── page.tsx                  # Landing page
│   └── globals.css               # Global styles
│
├── components/                   # Reusable components
│   ├── ui/                       # Base UI components
│   │   ├── Button.tsx
│   │   ├── Input.tsx
│   │   ├── Card.tsx
│   │   ├── Modal.tsx
│   │   └── Dropdown.tsx
│   │
│   ├── features/                 # Feature-specific components
│   │   ├── transactions/
│   │   │   ├── TransactionList.tsx
│   │   │   ├── TransactionForm.tsx
│   │   │   └── TransactionCard.tsx
│   │   ├── budgets/
│   │   │   ├── BudgetOverview.tsx
│   │   │   ├── BudgetChart.tsx
│   │   │   └── BudgetForm.tsx
│   │   └── goals/
│   │       ├── GoalProgress.tsx
│   │       └── GoalForm.tsx
│   │
│   └── layout/                   # Layout components
│       ├── Header.tsx
│       ├── Sidebar.tsx
│       ├── Footer.tsx
│       └── Navigation.tsx
│
├── lib/                          # Utility libraries
│   ├── db.ts                     # Prisma client instance
│   ├── auth.ts                   # Auth helpers
│   ├── api.ts                    # API client
│   └── utils.ts                  # Helper functions
│
├── hooks/                        # Custom React hooks
│   ├── useTransactions.ts
│   ├── useBudgets.ts
│   ├── useAuth.ts
│   └── useDebounce.ts
│
├── types/                        # TypeScript types
│   ├── index.ts
│   ├── api.ts
│   └── models.ts
│
└── store/                        # State management
    ├── authStore.ts              # Auth state (Zustand)
    ├── budgetStore.ts            # Budget state
    └── uiStore.ts                # UI state (theme, modals)
```

### Component Architecture

#### 1. Page Components (Routes)

```typescript
// app/(dashboard)/transactions/page.tsx
import { TransactionList } from '@/components/features/transactions/TransactionList'
import { useTransactions } from '@/hooks/useTransactions'

export default function TransactionsPage() {
  const { transactions, isLoading } = useTransactions()
  
  return (
    <div className="container">
      <h1>Transactions</h1>
      <TransactionList 
        transactions={transactions} 
        isLoading={isLoading} 
      />
    </div>
  )
}
```

#### 2. Feature Components

```typescript
// components/features/transactions/TransactionList.tsx
'use client'

import { Transaction } from '@/types'
import { TransactionCard } from './TransactionCard'

interface Props {
  transactions: Transaction[]
  isLoading: boolean
}

export function TransactionList({ transactions, isLoading }: Props) {
  if (isLoading) return <Skeleton />
  
  return (
    <div className="grid gap-4">
      {transactions.map(tx => (
        <TransactionCard key={tx.id} transaction={tx} />
      ))}
    </div>
  )
}
```

#### 3. Custom Hooks

```typescript
// hooks/useTransactions.ts
import { useQuery, useMutation } from '@tanstack/react-query'
import { api } from '@/lib/api'

export function useTransactions() {
  const { data, isLoading } = useQuery({
    queryKey: ['transactions'],
    queryFn: () => api.get('/api/transactions')
  })
  
  const addTransaction = useMutation({
    mutationFn: (data) => api.post('/api/transactions', data),
    onSuccess: () => {
      // Invalidate and refetch
      queryClient.invalidateQueries(['transactions'])
    }
  })
  
  return {
    transactions: data?.transactions || [],
    isLoading,
    addTransaction
  }
}
```

### State Management Strategy

```typescript
// store/budgetStore.ts - Zustand store
import { create } from 'zustand'

interface BudgetState {
  budgets: Budget[]
  selectedBudget: Budget | null
  setBudgets: (budgets: Budget[]) => void
  selectBudget: (id: string) => void
}

export const useBudgetStore = create<BudgetState>((set) => ({
  budgets: [],
  selectedBudget: null,
  setBudgets: (budgets) => set({ budgets }),
  selectBudget: (id) => set((state) => ({
    selectedBudget: state.budgets.find(b => b.id === id) || null
  }))
}))
```

### Data Flow

```
User Interaction
      ↓
React Component
      ↓
Event Handler
      ↓
Custom Hook (useTransactions)
      ↓
API Call (fetch)
      ↓
Backend API Route
      ↓
Database Query (Prisma)
      ↓
Response
      ↓
State Update (React Query cache)
      ↓
Component Re-render
      ↓
UI Update
```

---

## ⚙️ Backend Architecture (Next.js API Routes)

### API Structure

```
/api/
├── auth/[...nextauth]/route.ts   # Authentication endpoints
├── transactions/
│   ├── route.ts                  # GET, POST /api/transactions
│   └── [id]/route.ts             # GET, PUT, DELETE /api/transactions/:id
├── budgets/
│   ├── route.ts                  # Budget CRUD
│   └── [id]/route.ts
├── categories/
│   └── route.ts                  # Category management
├── goals/
│   ├── route.ts
│   └── [id]/route.ts
├── dashboard/
│   └── route.ts                  # Dashboard summary data
└── reports/
    ├── monthly/route.ts          # Monthly report
    └── yearly/route.ts           # Yearly report
```

### API Route Example

```typescript
// app/api/transactions/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { getServerSession } from 'next-auth'
import { prisma } from '@/lib/db'
import { z } from 'zod'

// Validation schema
const TransactionSchema = z.object({
  amount: z.number().positive(),
  date: z.string().datetime(),
  categoryId: z.string(),
  description: z.string().optional(),
  type: z.enum(['income', 'expense'])
})

// GET /api/transactions
export async function GET(request: NextRequest) {
  const session = await getServerSession()
  
  if (!session?.user?.id) {
    return NextResponse.json(
      { error: 'Unauthorized' },
      { status: 401 }
    )
  }
  
  const { searchParams } = new URL(request.url)
  const startDate = searchParams.get('startDate')
  const endDate = searchParams.get('endDate')
  
  const transactions = await prisma.transaction.findMany({
    where: {
      userId: session.user.id,
      date: {
        gte: startDate ? new Date(startDate) : undefined,
        lte: endDate ? new Date(endDate) : undefined
      }
    },
    include: {
      category: true
    },
    orderBy: {
      date: 'desc'
    }
  })
  
  return NextResponse.json({ transactions })
}

// POST /api/transactions
export async function POST(request: NextRequest) {
  const session = await getServerSession()
  
  if (!session?.user?.id) {
    return NextResponse.json(
      { error: 'Unauthorized' },
      { status: 401 }
    )
  }
  
  try {
    const body = await request.json()
    const data = TransactionSchema.parse(body)
    
    const transaction = await prisma.transaction.create({
      data: {
        ...data,
        userId: session.user.id,
        date: new Date(data.date)
      },
      include: {
        category: true
      }
    })
    
    return NextResponse.json(
      { transaction },
      { status: 201 }
    )
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json(
        { error: 'Invalid data', details: error.errors },
        { status: 400 }
      )
    }
    
    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    )
  }
}
```

### Authentication Flow (NextAuth.js)

```typescript
// app/api/auth/[...nextauth]/route.ts
import NextAuth from 'next-auth'
import CredentialsProvider from 'next-auth/providers/credentials'
import { PrismaAdapter } from '@auth/prisma-adapter'
import { prisma } from '@/lib/db'
import bcrypt from 'bcryptjs'

const handler = NextAuth({
  adapter: PrismaAdapter(prisma),
  providers: [
    CredentialsProvider({
      name: 'Credentials',
      credentials: {
        email: { label: "Email", type: "email" },
        password: { label: "Password", type: "password" }
      },
      async authorize(credentials) {
        if (!credentials?.email || !credentials?.password) {
          return null
        }
        
        const user = await prisma.user.findUnique({
          where: { email: credentials.email }
        })
        
        if (!user || !user.password) {
          return null
        }
        
        const isValid = await bcrypt.compare(
          credentials.password,
          user.password
        )
        
        if (!isValid) {
          return null
        }
        
        return {
          id: user.id,
          email: user.email,
          name: user.name
        }
      }
    })
  ],
  session: {
    strategy: 'jwt'
  },
  pages: {
    signIn: '/login',
    signOut: '/logout',
    error: '/auth/error'
  },
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.id = user.id
      }
      return token
    },
    async session({ session, token }) {
      if (session.user) {
        session.user.id = token.id as string
      }
      return session
    }
  }
})

export { handler as GET, handler as POST }
```

### Business Logic Layer

```typescript
// lib/services/budgetService.ts
import { prisma } from '@/lib/db'

export class BudgetService {
  static async calculateBudgetStatus(userId: string, month: string) {
    // Get user's budget for the month
    const budget = await prisma.budget.findFirst({
      where: { userId, month: new Date(month) },
      include: { categories: true }
    })
    
    if (!budget) return null
    
    // Get actual spending for the month
    const transactions = await prisma.transaction.findMany({
      where: {
        userId,
        date: {
          gte: new Date(month),
          lt: new Date(new Date(month).setMonth(new Date(month).getMonth() + 1))
        },
        type: 'expense'
      }
    })
    
    // Calculate totals per category
    const categoryTotals = transactions.reduce((acc, tx) => {
      acc[tx.categoryId] = (acc[tx.categoryId] || 0) + Number(tx.amount)
      return acc
    }, {} as Record<string, number>)
    
    // Compare budget vs actual
    const status = budget.categories.map(cat => ({
      categoryId: cat.id,
      categoryName: cat.name,
      budgeted: Number(cat.amount),
      actual: categoryTotals[cat.id] || 0,
      difference: Number(cat.amount) - (categoryTotals[cat.id] || 0),
      percentUsed: (categoryTotals[cat.id] || 0) / Number(cat.amount) * 100
    }))
    
    return {
      budget,
      status,
      totalBudgeted: budget.categories.reduce((sum, cat) => sum + Number(cat.amount), 0),
      totalActual: Object.values(categoryTotals).reduce((sum, val) => sum + val, 0)
    }
  }
  
  static async getSavingsRate(userId: string, startDate: Date, endDate: Date) {
    const income = await prisma.transaction.aggregate({
      where: {
        userId,
        type: 'income',
        date: { gte: startDate, lte: endDate }
      },
      _sum: { amount: true }
    })
    
    const expenses = await prisma.transaction.aggregate({
      where: {
        userId,
        type: 'expense',
        date: { gte: startDate, lte: endDate }
      },
      _sum: { amount: true }
    })
    
    const totalIncome = Number(income._sum.amount || 0)
    const totalExpenses = Number(expenses._sum.amount || 0)
    const savings = totalIncome - totalExpenses
    const savingsRate = totalIncome === 0 ? 0 : (savings / totalIncome) * 100
    
    return {
      totalIncome,
      totalExpenses,
      savings,
      savingsRate
    }
  }
}
```

---

## 💾 Database Architecture (PostgreSQL + Prisma)

### Database Schema

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// User model
model User {
  id            String          @id @default(cuid())
  email         String          @unique
  name          String?
  password      String?         // Hashed
  emailVerified DateTime?
  image         String?
  createdAt     DateTime        @default(now())
  updatedAt     DateTime        @updatedAt
  
  // Relations
  accounts      Account[]
  sessions      Session[]
  transactions  Transaction[]
  categories    ExpenseCategory[]
  budgets       Budget[]
  goals         SavingsGoal[]
  incomeSources IncomeSource[]
  
  @@map("users")
}

// NextAuth models
model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String?
  access_token      String?
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String?
  session_state     String?
  
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@unique([provider, providerAccountId])
  @@map("accounts")
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@map("sessions")
}

// Income source
model IncomeSource {
  id        String   @id @default(cuid())
  userId    String
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  name      String
  amount    Decimal  @db.Decimal(10, 2)
  frequency String   // monthly, biweekly, weekly
  isActive  Boolean  @default(true)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  @@index([userId])
  @@map("income_sources")
}

// Expense category
model ExpenseCategory {
  id              String        @id @default(cuid())
  userId          String
  user            User          @relation(fields: [userId], references: [id], onDelete: Cascade)
  name            String
  parentCategory  String?       // For sub-categories
  budgetedAmount  Decimal       @db.Decimal(10, 2)
  color           String?       // For UI
  icon            String?       // For UI
  isActive        Boolean       @default(true)
  createdAt       DateTime      @default(now())
  updatedAt       DateTime      @updatedAt
  
  // Relations
  transactions    Transaction[]
  budgetItems     BudgetItem[]
  
  @@index([userId])
  @@index([userId, parentCategory])
  @@map("expense_categories")
}

// Transaction
model Transaction {
  id          String           @id @default(cuid())
  userId      String
  user        User             @relation(fields: [userId], references: [id], onDelete: Cascade)
  date        DateTime
  amount      Decimal          @db.Decimal(10, 2)
  description String?
  type        TransactionType  // income or expense
  categoryId  String?
  category    ExpenseCategory? @relation(fields: [categoryId], references: [id], onDelete: SetNull)
  recurring   Boolean          @default(false)
  receiptUrl  String?          // For receipt photos
  tags        String[]         // For flexible categorization
  createdAt   DateTime         @default(now())
  updatedAt   DateTime         @updatedAt
  
  @@index([userId])
  @@index([userId, date])
  @@index([userId, type])
  @@index([categoryId])
  @@map("transactions")
}

enum TransactionType {
  income
  expense
}

// Budget
model Budget {
  id        String       @id @default(cuid())
  userId    String
  user      User         @relation(fields: [userId], references: [id], onDelete: Cascade)
  name      String
  month     DateTime     // First day of the month
  items     BudgetItem[]
  createdAt DateTime     @default(now())
  updatedAt DateTime     @updatedAt
  
  @@unique([userId, month])
  @@index([userId])
  @@map("budgets")
}

model BudgetItem {
  id         String          @id @default(cuid())
  budgetId   String
  budget     Budget          @relation(fields: [budgetId], references: [id], onDelete: Cascade)
  categoryId String
  category   ExpenseCategory @relation(fields: [categoryId], references: [id], onDelete: Cascade)
  amount     Decimal         @db.Decimal(10, 2)
  createdAt  DateTime        @default(now())
  updatedAt  DateTime        @updatedAt
  
  @@unique([budgetId, categoryId])
  @@index([budgetId])
  @@map("budget_items")
}

// Savings goal
model SavingsGoal {
  id            String   @id @default(cuid())
  userId        String
  user          User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  name          String
  targetAmount  Decimal  @db.Decimal(10, 2)
  currentAmount Decimal  @db.Decimal(10, 2) @default(0)
  deadline      DateTime?
  priority      Int      @default(1)
  color         String?
  icon          String?
  isCompleted   Boolean  @default(false)
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
  
  @@index([userId])
  @@index([userId, isCompleted])
  @@map("savings_goals")
}
```

### Database Indexes

```sql
-- Performance optimization indexes
CREATE INDEX idx_transactions_user_date ON transactions(user_id, date DESC);
CREATE INDEX idx_transactions_user_type ON transactions(user_id, type);
CREATE INDEX idx_transactions_category ON transactions(category_id);
CREATE INDEX idx_transactions_date_range ON transactions(date) WHERE date >= NOW() - INTERVAL '1 year';

CREATE INDEX idx_budgets_user_month ON budgets(user_id, month DESC);
CREATE INDEX idx_budget_items_budget ON budget_items(budget_id);

CREATE INDEX idx_categories_user_active ON expense_categories(user_id, is_active);
CREATE INDEX idx_goals_user_completed ON savings_goals(user_id, is_completed);
```

### Common Queries

```typescript
// Get user's dashboard summary
const getDashboardSummary = async (userId: string) => {
  const now = new Date()
  const firstDayOfMonth = new Date(now.getFullYear(), now.getMonth(), 1)
  const lastDayOfMonth = new Date(now.getFullYear(), now.getMonth() + 1, 0)
  
  // Run queries in parallel
  const [income, expenses, budget, goals] = await Promise.all([
    // Total income this month
    prisma.transaction.aggregate({
      where: {
        userId,
        type: 'income',
        date: { gte: firstDayOfMonth, lte: lastDayOfMonth }
      },
      _sum: { amount: true }
    }),
    
    // Total expenses this month
    prisma.transaction.aggregate({
      where: {
        userId,
        type: 'expense',
        date: { gte: firstDayOfMonth, lte: lastDayOfMonth }
      },
      _sum: { amount: true }
    }),
    
    // Current month budget
    prisma.budget.findFirst({
      where: {
        userId,
        month: firstDayOfMonth
      },
      include: {
        items: {
          include: { category: true }
        }
      }
    }),
    
    // Active savings goals
    prisma.savingsGoal.findMany({
      where: {
        userId,
        isCompleted: false
      },
      orderBy: { priority: 'asc' }
    })
  ])
  
  return {
    totalIncome: Number(income._sum.amount || 0),
    totalExpenses: Number(expenses._sum.amount || 0),
    netIncome: Number(income._sum.amount || 0) - Number(expenses._sum.amount || 0),
    budget,
    goals
  }
}
```

---

## 🔐 Security Architecture

### Authentication & Authorization

```
┌─────────────────┐
│  User Logs In   │
└────────┬────────┘
         │
         ▼
┌─────────────────────────┐
│  NextAuth.js            │
│  - Verifies credentials │
│  - Creates session      │
│  - Issues JWT token     │
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│  JWT Token (HTTP-only cookie)   │
│  Contains: userId, email, exp   │
└────────┬────────────────────────┘
         │
         ▼
┌─────────────────────────┐
│  Protected API Request  │
│  - Cookie sent auto     │
│  - Verified on server   │
└────────┬────────────────┘
         │
         ▼
┌──────────────────────────┐
│  Middleware Checks       │
│  - Valid token?          │
│  - Not expired?          │
│  - User exists?          │
└────────┬─────────────────┘
         │
         ▼
┌─────────────────────────┐
│  Access Granted         │
│  - User data returned   │
└─────────────────────────┘
```

### Security Best Practices Implemented

```typescript
// 1. Row-Level Security (check user owns data)
export async function getTransaction(userId: string, transactionId: string) {
  const transaction = await prisma.transaction.findFirst({
    where: {
      id: transactionId,
      userId: userId  // CRITICAL: Prevent access to other users' data
    }
  })
  
  if (!transaction) {
    throw new Error('Transaction not found or access denied')
  }
  
  return transaction
}

// 2. Input Validation (Zod schemas)
const TransactionSchema = z.object({
  amount: z.number().positive().max(1000000),
  date: z.string().datetime(),
  type: z.enum(['income', 'expense']),
  description: z.string().max(500).optional()
})

// 3. Rate Limiting (middleware)
import { rateLimit } from '@/lib/rate-limit'

export async function POST(request: NextRequest) {
  const limiter = rateLimit({
    interval: 60 * 1000, // 1 minute
    uniqueTokenPerInterval: 500
  })
  
  try {
    await limiter.check(10, 'API_LIMIT') // 10 requests per minute
  } catch {
    return NextResponse.json(
      { error: 'Rate limit exceeded' },
      { status: 429 }
    )
  }
  
  // Process request...
}

// 4. SQL Injection Prevention (Prisma ORM)
// ✅ SAFE - Prisma parameterizes queries
const user = await prisma.user.findUnique({
  where: { email: userEmail }
})

// ❌ NEVER DO THIS (raw SQL without parameterization)
// await prisma.$executeRaw`SELECT * FROM users WHERE email = '${userEmail}'`

// 5. XSS Prevention (React auto-escapes)
// ✅ SAFE - React escapes by default
<div>{userInput}</div>

// ❌ DANGEROUS - Never use dangerouslySetInnerHTML with user input
<div dangerouslySetInnerHTML={{ __html: userInput }} />
```

---

## 🚀 Deployment Architecture (GCP)

### Infrastructure Components

```
┌──────────────────────────────────────────────────────────┐
│                      INTERNET                             │
└───────────────────────┬──────────────────────────────────┘
                        │
                        │ HTTPS
                        ▼
        ┌────────────────────────────────────┐
        │     Cloud Load Balancer            │
        │  • SSL/TLS Termination             │
        │  • DDoS Protection (Cloud Armor)   │
        │  • CDN (Cloud CDN)                 │
        └────────────────┬───────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                   CLOUD RUN                              │
│  ┌────────────────────────────────────────────────────┐ │
│  │  Container Instance 1  │  Container Instance 2     │ │
│  │  (Next.js App)         │  (Next.js App)            │ │
│  │  • Auto-scaling         │  • Load balanced          │ │
│  │  • 512MB RAM            │  • Health checked         │ │
│  └────────────────────────────────────────────────────┘ │
└──────────────────┬──────────────────┬───────────────────┘
                   │                  │
        VPC        │                  │       Private
        Connector  │                  │       Network
                   ▼                  ▼
    ┌──────────────────┐   ┌──────────────────┐
    │   CLOUD SQL      │   │  SECRET MANAGER  │
    │   (PostgreSQL)   │   │  • DB Password   │
    │   • Private IP   │   │  • Auth Secret   │
    │   • Backups      │   │  • API Keys      │
    └──────────────────┘   └──────────────────┘
```

### Container Configuration

```dockerfile
# Optimized Dockerfile for Cloud Run
FROM node:18-alpine AS base

FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci

FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Generate Prisma Client
RUN npx prisma generate

# Build Next.js app
ENV NEXT_TELEMETRY_DISABLED 1
RUN npm run build

FROM base AS runner
WORKDIR /app

ENV NODE_ENV production
ENV NEXT_TELEMETRY_DISABLED 1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# Copy necessary files
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
COPY --from=builder /app/prisma ./prisma
COPY --from=builder /app/node_modules/.prisma ./node_modules/.prisma

USER nextjs

EXPOSE 8080
ENV PORT 8080
ENV HOSTNAME "0.0.0.0"

CMD ["node", "server.js"]
```

### Cloud Run Configuration

```bash
# Deploy with optimal settings
gcloud run deploy budget-app \
  --image=IMAGE_URL \
  --platform=managed \
  --region=us-central1 \
  --allow-unauthenticated \
  --memory=512Mi \
  --cpu=1 \
  --min-instances=0 \       # Scale to zero when idle (cost savings)
  --max-instances=10 \      # Limit max scaling
  --timeout=300 \           # 5 minute timeout
  --concurrency=80 \        # 80 requests per instance
  --cpu-throttling \        # Throttle CPU when no requests
  --port=8080
```

---

## 📊 Monitoring & Observability

### Logging Strategy

```typescript
// lib/logger.ts
export class Logger {
  static info(message: string, metadata?: Record<string, any>) {
    console.log(JSON.stringify({
      level: 'info',
      message,
      timestamp: new Date().toISOString(),
      ...metadata
    }))
  }
  
  static error(message: string, error: Error, metadata?: Record<string, any>) {
    console.error(JSON.stringify({
      level: 'error',
      message,
      error: {
        message: error.message,
        stack: error.stack
      },
      timestamp: new Date().toISOString(),
      ...metadata
    }))
  }
}

// Usage in API route
import { Logger } from '@/lib/logger'

export async function POST(request: NextRequest) {
  try {
    // ... process request
    Logger.info('Transaction created', {
      userId: session.user.id,
      amount: data.amount
    })
  } catch (error) {
    Logger.error('Failed to create transaction', error as Error, {
      userId: session.user.id
    })
    throw error
  }
}
```

### Performance Monitoring

```typescript
// Measure database query performance
async function measureQuery<T>(
  name: string,
  query: () => Promise<T>
): Promise<T> {
  const start = performance.now()
  
  try {
    const result = await query()
    const duration = performance.now() - start
    
    Logger.info('Query completed', {
      queryName: name,
      duration: `${duration}ms`
    })
    
    return result
  } catch (error) {
    const duration = performance.now() - start
    
    Logger.error('Query failed', error as Error, {
      queryName: name,
      duration: `${duration}ms`
    })
    
    throw error
  }
}

// Usage
const transactions = await measureQuery(
  'getTransactions',
  () => prisma.transaction.findMany({ where: { userId } })
)
```

---

## 🎯 Performance Optimization

### Caching Strategy

```typescript
// 1. Next.js Route Caching
export const revalidate = 3600 // Cache for 1 hour

export async function GET() {
  // This response is cached
  const data = await fetchData()
  return Response.json(data)
}

// 2. React Query Client-Side Caching
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 5 minutes
      cacheTime: 10 * 60 * 1000 // 10 minutes
    }
  }
})

// 3. Database Query Optimization
// Use indexes, select only needed fields
const transactions = await prisma.transaction.findMany({
  where: { userId },
  select: {
    id: true,
    amount: true,
    date: true,
    category: {
      select: {
        name: true,
        color: true
      }
    }
  },
  take: 50 // Limit results
})
```

### Code Splitting

```typescript
// Dynamic imports for heavy components
const ChartComponent = dynamic(() => import('@/components/Chart'), {
  loading: () => <Skeleton />,
  ssr: false // Don't render on server
})

// Route-based code splitting (automatic in Next.js)
// Each page is automatically split into its own bundle
```

---

## 📖 Summary

This architecture provides:

- ✅ **Scalable**: Auto-scales from 0 to many instances
- ✅ **Secure**: Auth, validation, row-level security
- ✅ **Performant**: Caching, optimized queries, code splitting
- ✅ **Maintainable**: Clean separation of concerns, TypeScript
- ✅ **Cost-Effective**: Pay only for what you use
- ✅ **Observable**: Comprehensive logging and monitoring

The system is designed to grow with your needs while keeping costs low initially.

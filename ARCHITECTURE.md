# FlintCal - Architecture Diagram

This document provides visual diagrams of how FlintCal works.

---

## 🏗️ Application Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         USER BROWSER                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                     App.vue (Root)                         │  │
│  │                                                            │  │
│  │  ┌──────────────────────────────────────────────────────┐ │  │
│  │  │  Header (Logo + Title + Date Range)                  │ │  │
│  │  └──────────────────────────────────────────────────────┘ │  │
│  │                                                            │  │
│  │  ┌──────────────────────────────────────────────────────┐ │  │
│  │  │  UserSelector.vue                                     │ │  │
│  │  │  [Flint] [Bryan] [Leslie] [Molly]  ← User Chips     │ │  │
│  │  └──────────────────────────────────────────────────────┘ │  │
│  │                                                            │  │
│  │  ┌──────────────────────────────────────────────────────┐ │  │
│  │  │  Calendar.vue                                         │ │  │
│  │  │                                                        │ │  │
│  │  │        October 2025         ← Month Header            │ │  │
│  │  │     [<]            [>]      ← Navigation              │ │  │
│  │  │                                                        │ │  │
│  │  │   S  M  T  W  T  F  S                                │ │  │
│  │  │      1  2  3  4  5  6                                │ │  │
│  │  │   7  8  9 10 11 12 13      ← Calendar Grid          │ │  │
│  │  │  14 15 16 17 18 19 20      ← Color-coded dates      │ │  │
│  │  │  21 22 23 24 25 26 27                                │ │  │
│  │  │  28 29 30 31                                          │ │  │
│  │  │                                                        │ │  │
│  │  │  Legend: 🔵 Flint 🟢 Bryan 🟠 Leslie 🟣 Molly       │ │  │
│  │  └──────────────────────────────────────────────────────┘ │  │
│  │                                                            │  │
│  │  ┌──────────────────────────────────────────────────────┐ │  │
│  │  │  Footer (Copyright + Connection Status)              │ │  │
│  │  └──────────────────────────────────────────────────────┘ │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
                            ↕️
                    (Real-time Sync)
                            ↕️
┌─────────────────────────────────────────────────────────────────┐
│                      SUPABASE CLOUD                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  PostgreSQL Database:                                            │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  user_availability table                                 │   │
│  │  ─────────────────────────────────────────────────────  │   │
│  │  id | user_name | selected_date | created_at | updated  │   │
│  │  ─────────────────────────────────────────────────────  │   │
│  │  1  | Flint...  | 2025-10-15   | ...        | ...       │   │
│  │  2  | Bryan...  | 2025-10-15   | ...        | ...       │   │
│  │  3  | Flint...  | 2025-10-20   | ...        | ...       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                   │
│  Real-time Subscriptions: WebSocket connections for live sync   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Data Flow Diagram

```
USER INTERACTION                    APPLICATION LOGIC                 DATABASE
─────────────────                   ──────────────────                ────────

1. User clicks date
      │
      ├──> Calendar.vue
      │       │
      │       ├──> onDayClick()
      │       │       │
      │       │       ├──> Get active user (useUsers)
      │       │       │
      │       │       ├──> Convert date (useDates)
      │       │       │
      │       │       └──> Emit event to App.vue
      │                       │
      │                       ├──> handleDateToggle()
      │                       │       │
      │                       │       ├──> useSupabase.toggleUserDate()
      │                       │       │       │
      │                       │       │       └──> Supabase INSERT/DELETE ──> Database
      │                       │       │                                          │
      │                       │       └──> Update local state (optimistic)       │
      │                       │                                                  │
      │                       └──> Show sync indicator                           │
      │                                                                          │
      └────────────────────────────────────────────────────────────────────────┘
                                                                                 │
2. Database changes                                                              │
      │                                                                          │
      ◄────────────────────────────────────────────────────────────────────────┘
      │
      ├──> Supabase Real-time
      │       │
      │       ├──> WebSocket notification
      │       │
      │       └──> subscribeToChanges() callback
      │               │
      │               ├──> handleRealtimeUpdate()
      │               │       │
      │               │       ├──> loadAllAvailability()
      │               │       │
      │               │       └──> Update UI (all browsers!)
      │               │
      │               └──> Show notification
      │
      └──> Calendar updates automatically
```

---

## 📦 Component Hierarchy

```
main.js
  │
  ├──> Creates Vue app
  ├──> Registers Vuetify
  ├──> Registers V-Calendar
  │
  └──> Mounts App.vue
         │
         ├──> UserSelector.vue
         │      │
         │      └──> Uses: useUsers composable
         │
         ├──> Calendar.vue
         │      │
         │      ├──> Uses: useUsers composable
         │      ├──> Uses: useDates composable
         │      └──> Uses: config.js (date range)
         │
         ├──> LoadingState.vue
         │
         └──> Uses: useSupabase composable
                  │
                  └──> Connects to: Supabase
```

---

## 🔧 Composables Structure

```
┌─────────────────────────────────────────────────────────────┐
│                    COMPOSABLES                               │
│                 (Reusable Logic)                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  useSupabase.js                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  • loadAllAvailability()      - Get all data         │  │
│  │  • getUserDates()             - Get user's dates     │  │
│  │  • addUserDate()              - Add a date           │  │
│  │  • removeUserDate()           - Remove a date        │  │
│  │  • toggleUserDate()           - Add or remove        │  │
│  │  • subscribeToChanges()       - Real-time listener   │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  useUsers.js                                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  • users                      - All users list       │  │
│  │  • activeUser                 - Current user         │  │
│  │  • setActiveUser()            - Change user          │  │
│  │  • getUserByName()            - Find user            │  │
│  │  • getUserColor()             - Get user color       │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  useDates.js                                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  • PureDate class             - Timezone-safe dates  │  │
│  │  • dateToString()             - Date → YYYY-MM-DD    │  │
│  │  • stringToDate()             - YYYY-MM-DD → Date    │  │
│  │  • getToday()                 - Today's date         │  │
│  │  • formatDate()               - Pretty formatting    │  │
│  │  • getMonthName()             - Month name           │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 🗂️ Configuration Flow

```
config.js (Your edits)
    │
    ├──> users array
    │      │
    │      ├──> Used by: useUsers.js
    │      ├──> Used by: UserSelector.vue
    │      └──> Used by: Calendar.vue
    │
    ├──> dateRange object
    │      │
    │      ├──> Used by: useDates.js
    │      ├──> Used by: Calendar.vue
    │      └──> Used by: App.vue (header display)
    │
    ├──> ui settings
    │      │
    │      └──> Used by: main.js (Vuetify theme)
    │
    └──> features flags
           │
           └──> Used by: App.vue (feature enabling)
```

---

## 🔄 Real-Time Sync Flow

```
Browser A                    Supabase                    Browser B
─────────                    ────────                    ─────────

1. User clicks date
      │
      ├──> Save to DB ──────────> Database UPDATE
      │                                 │
      └──> Update UI (instant)          │
                                        │
                                        ├──> Real-time notification
                                        │
                                        └──────────────────> Receive update
                                                                   │
                                                                   ├──> Load new data
                                                                   │
                                                                   └──> Update UI
                                                                   
Result: Both browsers show the same data!
```

---

## 📊 Database Schema Relationships

```
user_availability table
┌──────────────────────────────────────────────────────────┐
│  Column         Type          Constraints                 │
├──────────────────────────────────────────────────────────┤
│  id             BIGSERIAL     PRIMARY KEY                 │
│  user_name      TEXT          NOT NULL                    │
│  selected_date  DATE          NOT NULL                    │
│  created_at     TIMESTAMPTZ   DEFAULT NOW()               │
│  updated_at     TIMESTAMPTZ   DEFAULT NOW()               │
│                                                            │
│  UNIQUE(user_name, selected_date) ← No duplicate dates    │
└──────────────────────────────────────────────────────────┘
        │
        ├──> Index: idx_user_availability_user_name
        ├──> Index: idx_user_availability_selected_date
        └──> Index: idx_user_availability_composite

Example data:
┌────┬─────────────────┬───────────────┬────────────────────┐
│ id │ user_name       │ selected_date │ created_at         │
├────┼─────────────────┼───────────────┼────────────────────┤
│ 1  │ Flint & Maryam  │ 2025-10-15   │ 2025-10-07 10:30  │
│ 2  │ Bryan & Marlene │ 2025-10-15   │ 2025-10-07 11:45  │
│ 3  │ Flint & Maryam  │ 2025-10-20   │ 2025-10-07 14:20  │
└────┴─────────────────┴───────────────┴────────────────────┘
        ↑                      ↑
    Same user           Different dates → Both entries exist
    
    Different users     Same date → Both entries exist (multi-select)
```

---

## 🎨 Color Coding System

```
User Selection                Calendar Display
──────────────                ────────────────

Flint & Maryam
  color: #2196F3 (Blue) ────→ Date background: Light blue
  displayColor: #1976D2 ────→ User chip: Dark blue

Bryan & Marlene
  color: #4CAF50 (Green) ───→ Date background: Light green
  displayColor: #388E3C ────→ User chip: Dark green

Leslie & Manny
  color: #FF9800 (Orange) ──→ Date background: Light orange
  displayColor: #F57C00 ────→ User chip: Dark orange

Molly & Jay
  color: #9C27B0 (Purple) ──→ Date background: Light purple
  displayColor: #7B1FA2 ────→ User chip: Dark purple

Multiple users on same date:
  → Colors blend with opacity
  → All colors visible
```

---

## 🔐 Security Architecture

```
Browser                      Supabase                    Database
───────                      ────────                    ────────

User request
    │
    ├──> Uses: .env credentials
    │       │
    │       ├──> VITE_SUPABASE_URL
    │       └──> VITE_SUPABASE_ANON_KEY
    │
    └──> API request ────────────> Check RLS policy
                                        │
                                        ├──> Policy: "Allow all"
                                        │    USING (true)
                                        │    WITH CHECK (true)
                                        │
                                        └──> Execute query ──> Database
                                                                   │
                                        Response <─────────────────┘
                                            │
Browser <───────────────────────────────────┘

Note: RLS = Row Level Security (Supabase feature)
      Public access is intentional for this use case
```

---

## 📱 Responsive Design Breakpoints

```
Desktop (1200px+)               Tablet (768-1199px)          Mobile (320-767px)
─────────────────               ───────────────────          ──────────────────

┌─────────────────┐             ┌──────────────┐            ┌─────────┐
│  Header + Logo  │             │ Header + Logo│            │ Header  │
├─────────────────┤             ├──────────────┤            ├─────────┤
│ [U1][U2][U3][U4]│             │ [U1][U2]     │            │ [U1]    │
│ Horizontal chips│             │ [U3][U4]     │            │ [U2]    │
├─────────────────┤             │ Wrapped chips│            │ [U3]    │
│                 │             ├──────────────┤            │ [U4]    │
│   Large         │             │   Medium     │            │ Stacked │
│   Calendar      │             │   Calendar   │            ├─────────┤
│   60px cells    │             │   50px cells │            │ Compact │
│                 │             │              │            │ Calendar│
│                 │             │              │            │ 40px    │
├─────────────────┤             ├──────────────┤            ├─────────┤
│  Footer         │             │ Footer       │            │ Footer  │
└─────────────────┘             └──────────────┘            └─────────┘
```

---

## 🚀 Deployment Architecture

```
Development                   Build                      Production
───────────                   ─────                      ──────────

Local files                   npm run build              Hosting Platform
    │                              │                            │
    ├──> src/                      ├──> Vite                    │
    ├──> components/               │      │                     │
    ├──> composables/              │      ├──> Transpile Vue    │
    ├──> config.js                 │      ├──> Bundle JS        │
    └──> .env                      │      ├──> Optimize CSS     │
                                   │      └──> Copy public/     │
                                   │                            │
                                   └──> dist/                   │
                                          │                     │
                                          ├──> index.html ──────┼──> Netlify
                                          ├──> assets/          │    Vercel
                                          │    ├── app.js       │    GitHub Pages
                                          │    ├── app.css      │    etc.
                                          │    └── logo.png     │
                                          │                     │
                                          └──> Deployed ────────┘
                                                    │
                                                    ↓
                                            https://your-app.com
```

---

## 🎯 User Journey Map

```
New User Arrives                   First Interaction              Regular Use
────────────────                   ─────────────────              ───────────

1. Visit URL                       1. Select user                 1. Visit site
      ↓                                  ↓                              ↓
2. See loading                     2. Click dates                 2. Auto-load data
      ↓                                  ↓                              ↓
3. Load all dates                  3. See confirmation            3. Select user
   from database                         ↓                              ↓
      ↓                            4. See date highlight           4. Update dates
4. Display calendar                      ↓                              ↓
      ↓                            5. Real-time notification       5. Real-time sync
5. Show all users'                       ↓                              ↓
   availability                    6. Understand system            6. Seamless experience
      ↓                                  ↓                              ↓
6. User can interact               7. Regular user!                ∞ Continue using
```

---

## 🧩 Module Dependencies

```
External Dependencies              Internal Dependencies
─────────────────────              ────────────────────

Vue 3                              config.js
  └──> Used by: All .vue files       ├──> useUsers.js
                                     ├──> useDates.js
Vuetify                              ├──> Calendar.vue
  └──> Used by: All components       └──> App.vue

V-Calendar                         useSupabase.js
  └──> Used by: Calendar.vue         └──> App.vue

Supabase JS                        useUsers.js
  └──> Used by: useSupabase.js       ├──> UserSelector.vue
                                     ├──> Calendar.vue
.env                                 └──> App.vue
  └──> Used by: useSupabase.js
                                   useDates.js
                                     ├──> Calendar.vue
                                     └──> App.vue
```

---

## 📈 Performance Optimization

```
Loading Performance                 Runtime Performance
───────────────────                 ───────────────────

1. Initial page load                1. User clicks date
      ↓                                   ↓
2. Load critical JS                 2. Optimistic update
   (Vue, Vuetify)                      (instant feedback)
      ↓                                   ↓
3. Load app code                    3. Background save
      ↓                                   ↓
4. Connect to Supabase              4. Database operation
      ↓                                   ↓
5. Load availability data           5. Real-time broadcast
      ↓                                   ↓
6. Render calendar                  6. Other browsers update
      ↓                                   ↓
✅ Interactive!                     ✅ Synced!

Optimizations:                      Optimizations:
- Vite code splitting              - Optimistic updates
- Tree shaking                     - Efficient re-renders
- CSS optimization                 - Debounced operations
- Asset compression                - Indexed database queries
```

---

**These diagrams show how FlintCal is architected at every level!**

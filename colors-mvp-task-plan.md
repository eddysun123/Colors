# Colors App MVP - Devin-Ready Implementation Plan

## Project Overview
A close friends feelings-sharing app where users:
1. Pick a color representing their mood
2. Choose a word describing their feeling  
3. Optionally explain why
4. View their group as a 6-slice ring showing everyone's latest colors
5. Send supportive messages via prefilled SMS/iMessage
6. Receive random daily nudges to check in

**Tech Stack:** Expo (React Native) + Supabase + TypeScript
**Repository:** https://github.com/eddysun123/Colors
**Timeline:** 1-2 weeks MVP
**Platform Priority:** iOS-first, Android-compatible

## TASK 1: Project Bootstrap & Setup

### 1.1 Initialize Expo Project
**Commands to execute:**
```bash
cd /home/ubuntu/repos/Colors
npx create-expo-app@latest . --template blank-typescript
```

**Required dependencies to add:**
```bash
npx expo install @supabase/supabase-js expo-contacts expo-sms expo-notifications react-native-svg expo-router zod
npm install @types/react @types/react-native
```

**Project structure to create:**
```
Colors/
├── app/                    # Expo Router pages
│   ├── (auth)/
│   │   ├── phone.tsx      # Phone input screen
│   │   └── otp.tsx        # OTP verification
│   ├── (tabs)/
│   │   ├── home.tsx       # Groups list
│   │   └── settings.tsx   # User settings
│   ├── groups/
│   │   └── [id].tsx       # Group ring view
│   ├── modals/
│   │   ├── logFeeling.tsx # Feeling log modal
│   │   └── createGroup.tsx # Create group flow
│   └── _layout.tsx        # Root layout
├── components/
│   ├── MoodRing.tsx       # SVG ring component
│   ├── SliceSheet.tsx     # Member detail modal
│   ├── ColorPicker.tsx    # Color selection
│   └── WordPicker.tsx     # Word selection
├── lib/
│   ├── supabase.ts        # Supabase client
│   ├── notifications.ts   # Push notification logic
│   ├── templates.ts       # Message templates
│   └── colorMap.ts        # Color/word mappings
├── supabase/
│   ├── functions/         # Edge functions
│   └── sql/              # Database schema
└── types/
    └── database.ts        # TypeScript types
```

**Verification:** 
- `npx expo start` runs without errors
- All dependencies install successfully
- Project structure matches specification

## TASK 2: Supabase Backend Setup

### 2.1 Database Schema Creation
**File to create:** `supabase/sql/001_schema.sql`
```sql
-- Enable UUID extension
create extension if not exists "uuid-ossp";

-- Users table
create table public.users (
  id uuid primary key default uuid_generate_v4(),
  phone text unique not null,
  display_name text not null,
  avatar_url text,
  created_at timestamptz default now()
);

-- Groups table  
create table public.groups (
  id uuid primary key default uuid_generate_v4(),
  owner_id uuid references public.users(id) on delete cascade,
  name text not null,
  emoji text,
  created_at timestamptz default now()
);

-- Group memberships (max 6 enforced by trigger)
create table public.group_members (
  group_id uuid references public.groups(id) on delete cascade,
  user_id uuid references public.users(id) on delete cascade,
  role text default 'member',
  joined_at timestamptz default now(),
  primary key (group_id, user_id)
);

-- Daily feelings (one per user per group per day)
create table public.feelings (
  id uuid primary key default uuid_generate_v4(),
  group_id uuid references public.groups(id) on delete cascade,
  user_id uuid references public.users(id) on delete cascade,
  day date not null,
  color text not null,
  word text not null,
  why text,
  created_at timestamptz default now(),
  unique (group_id, user_id, day)
);

-- Invite codes
create table public.invites (
  id uuid primary key default uuid_generate_v4(),
  group_id uuid references public.groups(id) on delete cascade,
  inviter_id uuid references public.users(id) on delete cascade,
  invite_code text unique not null,
  status text default 'pending',
  created_at timestamptz default now()
);

-- Push notification tokens
create table public.user_devices (
  user_id uuid references public.users(id) on delete cascade,
  expo_push_token text not null,
  updated_at timestamptz default now(),
  primary key (user_id, expo_push_token)
);

-- User notification settings
create table public.user_settings (
  user_id uuid references public.users(id) on delete cascade primary key,
  next_nudge_at timestamptz,
  notifications_enabled boolean default true,
  quiet_hours_start time default '22:00',
  quiet_hours_end time default '10:00'
);
```

### 2.2 Row Level Security Policies
**Commands to execute in Supabase SQL editor:**
```sql
-- Enable RLS
alter table public.users enable row level security;
alter table public.groups enable row level security;
alter table public.group_members enable row level security;
alter table public.feelings enable row level security;
alter table public.invites enable row level security;
alter table public.user_devices enable row level security;
alter table public.user_settings enable row level security;

-- Users can only see themselves
create policy "Users can view own profile" on public.users
  for select using (auth.uid() = id);

create policy "Users can update own profile" on public.users
  for update using (auth.uid() = id);

-- Group members can see their groups
create policy "Members can view their groups" on public.groups
  for select using (
    id in (
      select group_id from public.group_members 
      where user_id = auth.uid()
    )
  );

-- Group owners can update groups
create policy "Owners can update groups" on public.groups
  for update using (owner_id = auth.uid());

-- Members can see group membership
create policy "Members can view group membership" on public.group_members
  for select using (
    group_id in (
      select group_id from public.group_members 
      where user_id = auth.uid()
    )
  );

-- Feelings policies
create policy "Members can view group feelings" on public.feelings
  for select using (
    group_id in (
      select group_id from public.group_members 
      where user_id = auth.uid()
    )
  );

create policy "Users can insert own feelings" on public.feelings
  for insert with check (user_id = auth.uid());

create policy "Users can update own recent feelings" on public.feelings
  for update using (
    user_id = auth.uid() 
    and created_at > now() - interval '10 minutes'
  );
```

### 2.3 Edge Functions Setup
**Create these files:**

`supabase/functions/generate_daily_schedule/index.ts`:
```typescript
import { serve } from "https://deno.land/std@0.168.0/http/server.ts"
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

serve(async (req) => {
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL') ?? '',
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? ''
  )

  // Generate random time between 10:00-22:00 for each user
  const { data: users } = await supabase
    .from('users')
    .select('id')

  for (const user of users || []) {
    const randomHour = Math.floor(Math.random() * 12) + 10 // 10-21
    const randomMinute = Math.floor(Math.random() * 60)
    
    const tomorrow = new Date()
    tomorrow.setDate(tomorrow.getDate() + 1)
    tomorrow.setHours(randomHour, randomMinute, 0, 0)

    await supabase
      .from('user_settings')
      .upsert({
        user_id: user.id,
        next_nudge_at: tomorrow.toISOString()
      })
  }

  return new Response(JSON.stringify({ success: true }))
})
```

**Verification:**
- Supabase project created with phone auth enabled
- All tables created with proper relationships
- RLS policies active and tested
- Edge functions deployed and callable

### Task 1.3: Environment Configuration
- [ ] Set up environment variables:
  - `EXPO_PUBLIC_SUPABASE_URL`
  - `EXPO_PUBLIC_SUPABASE_ANON_KEY`
  - `SUPABASE_SERVICE_ROLE_KEY` (for Edge functions)
  - `EXPO_PUBLIC_APP_SCHEME`
- [ ] Configure Supabase client in app
- [ ] Set up deep linking configuration

## Phase 2: Authentication & User Management

### Task 2.1: Phone Authentication Flow
- [ ] Create `AuthPhoneScreen` component
- [ ] Implement phone number input with validation
- [ ] Implement OTP verification flow
- [ ] Handle user creation/login in Supabase
- [ ] Store user session and tokens
- [ ] Create user profile row on first signup

### Task 2.2: Onboarding Flow
- [ ] Request notification permissions
- [ ] Request contacts permissions
- [ ] Show privacy terms acceptance
- [ ] Register for push notifications
- [ ] Store Expo push token in `user_devices` table

## Phase 3: Groups & Membership

### Task 3.1: Create Group Flow
- [ ] Create `CreateGroupFlow` component
- [ ] Group name and emoji selection
- [ ] Contact picker integration (`expo-contacts`)
- [ ] Generate invite links via Edge function
- [ ] Send invites via SMS with prefilled message
- [ ] Handle group creation in database

### Task 3.2: Join Group Flow
- [ ] Handle deep link parsing for invite codes
- [ ] Validate invite codes
- [ ] Add user to group (max 6 members enforcement)
- [ ] Update invite status to 'accepted'
- [ ] Navigate to group view after joining

### Task 3.3: Group Management
- [ ] List user's groups on home screen
- [ ] Show group member count and status
- [ ] Basic group settings (leave group, etc.)

## Phase 4: Feelings Logging System

### Task 4.1: Color & Word System
- [ ] Create color palette (8 colors: BLUE, GREEN, YELLOW, ORANGE, RED, PURPLE, TEAL, GRAY)
- [ ] Create word mappings for each color
- [ ] Implement color picker UI component
- [ ] Implement word selection with chips/buttons
- [ ] Allow free-text word override

### Task 4.2: Feeling Log Modal
- [ ] Create `LogFeelingModal` component
- [ ] 3-step flow: Color → Word → Why (optional)
- [ ] Character limit for "why" field (140 chars)
- [ ] Validation and error handling
- [ ] Save feeling to database with today's date
- [ ] Prevent duplicate entries per day per group

### Task 4.3: Feeling Management
- [ ] Allow editing within 10 minutes (RLS policy)
- [ ] Show user's current feeling status
- [ ] Handle timezone considerations for "today"

## Phase 5: Group Ring Visualization

### Task 5.1: Mood Ring Component
- [ ] Create `MoodRing` component using `react-native-svg`
- [ ] Draw 6 equal slices in circular layout
- [ ] Color slices based on member's latest feeling
- [ ] Handle stale data (faded colors + diagonal hatch pattern)
- [ ] Make slices tappable with proper hit areas

### Task 5.2: Ring Data Management
- [ ] Fetch group members and their latest feelings
- [ ] Real-time updates via Supabase subscriptions
- [ ] Handle empty states (no feelings logged)
- [ ] Optimize for performance with 6 members max

### Task 5.3: Slice Detail View
- [ ] Create `SliceSheet` bottom modal
- [ ] Show member avatar, name, color, word, why, timestamp
- [ ] "Send support" button
- [ ] Handle member who hasn't logged today

## Phase 6: Support Messaging System

### Task 6.1: Message Templates
- [ ] Create template generation logic based on color/word combinations
- [ ] Handle different emotional states (sad, happy, stressed, etc.)
- [ ] Keep messages casual and supportive
- [ ] Allow user editing of generated templates

### Task 6.2: SMS Integration
- [ ] Integrate `expo-sms` for message composition
- [ ] Map group members to phone numbers
- [ ] Prefill recipient and message body
- [ ] Handle SMS availability and permissions
- [ ] User manually sends (iOS requirement)

### Task 6.3: Contact Mapping
- [ ] Match group members to user's contacts
- [ ] Handle cases where contact isn't found
- [ ] Confirm recipient before opening SMS

## Phase 7: Push Notifications & Nudges

### Task 7.1: Random Time Scheduling
- [ ] Edge function to calculate random time in 10am-10pm window
- [ ] Store `next_nudge_at` in user settings
- [ ] Handle timezone calculations
- [ ] Daily recalculation at midnight

### Task 7.2: Push Notification System
- [ ] Edge function to send daily pushes
- [ ] Check if user already logged today
- [ ] Different message for "log feeling" vs "check friends"
- [ ] Mark notifications as sent to avoid duplicates
- [ ] Handle notification delivery failures

### Task 7.3: Notification Handling
- [ ] Handle notification taps (deep link to log modal)
- [ ] Local notification fallback
- [ ] User preferences for quiet hours
- [ ] Notification permission management

## Phase 8: Settings & User Management

### Task 8.1: Settings Screen
- [ ] Toggle notifications on/off
- [ ] Show current groups and invite links
- [ ] Account deletion functionality
- [ ] Sign out functionality
- [ ] Privacy policy and terms links

### Task 8.2: Data Management
- [ ] Implement account deletion (cascade delete)
- [ ] Export user data functionality
- [ ] Handle data privacy requirements
- [ ] Clear local storage on sign out

## Phase 9: UI/UX Polish

### Task 9.1: Visual Design
- [ ] App icon and splash screen
- [ ] Color-blind accessibility (icons/patterns)
- [ ] Consistent typography and spacing
- [ ] Loading states and animations
- [ ] Error states and empty states

### Task 9.2: User Experience
- [ ] Smooth navigation transitions
- [ ] Haptic feedback for interactions
- [ ] Pull-to-refresh functionality
- [ ] Offline state handling
- [ ] Performance optimization

## Phase 10: Testing & Quality Assurance

### Task 10.1: Core Flow Testing
- [ ] Test complete user journey: signup → create group → invite → log feeling → view ring → send support
- [ ] Test with 6 test users in a group
- [ ] Verify all ring states: all logged, some stale, none logged
- [ ] Test push notifications at random times
- [ ] Test edge cases and error scenarios

### Task 10.2: Device & Platform Testing
- [ ] iOS testing (primary platform)
- [ ] Android compatibility testing
- [ ] Different screen sizes and orientations
- [ ] Permission handling across platforms
- [ ] Deep link handling

### Task 10.3: Performance & Security
- [ ] Database query optimization
- [ ] Real-time subscription performance
- [ ] Security audit of RLS policies
- [ ] Rate limiting for Edge functions
- [ ] Error logging and monitoring

## Phase 11: Deployment & Distribution

### Task 11.1: iOS Deployment
- [ ] Configure App Store metadata
- [ ] Create app screenshots and descriptions
- [ ] Set up TestFlight for internal testing
- [ ] Submit for App Store review
- [ ] Handle App Store feedback

### Task 11.2: Android Deployment (Optional for MVP)
- [ ] Configure Google Play metadata
- [ ] Internal testing track setup
- [ ] Play Store submission
- [ ] Handle Play Store feedback

### Task 11.3: Backend Deployment
- [ ] Deploy Supabase Edge functions
- [ ] Set up production environment variables
- [ ] Configure monitoring and alerts
- [ ] Set up backup and recovery procedures

## Phase 12: Analytics & Monitoring

### Task 12.1: Analytics Implementation
- [ ] Implement key events tracking:
  - `signup_complete`
  - `group_created`
  - `invite_sent`
  - `feeling_logged`
  - `slice_opened`
  - `support_initiated`
  - `push_received`
  - `push_opened`

### Task 12.2: KPI Monitoring
- [ ] Track D1/D7 retention rates
- [ ] Monitor daily logging rates
- [ ] Track support messages sent per user per week
- [ ] Monitor invite conversion rates
- [ ] Set up alerting for critical metrics

## Success Criteria (Definition of Done)

A new user can:
1. ✅ Sign up with phone OTP
2. ✅ Create a group with name and emoji
3. ✅ Invite 2+ friends via SMS
4. ✅ Log a daily feeling (color + word + optional why)
5. ✅ See group ring with all members' colors
6. ✅ Tap a slice to see friend's details
7. ✅ Send prefilled supportive message via SMS
8. ✅ Receive random daily push notification
9. ✅ Data is private to group members only
10. ✅ User can delete their account and data

## Risk Mitigation

- **SMS Auto-send Limitation**: Use compose sheet with prefilled body (iOS requirement)
- **Contact Mapping**: Ask user to confirm recipient before composing
- **Notification Delivery**: Rely on Expo; implement local fallback
- **Sensitive Content**: Keep private to groups; clear disclaimers; easy deletion
- **Performance**: Limit to 6 members per group; optimize real-time updates

## Post-MVP Roadmap

1. iMessage App Extension for inline interactions
2. Agentic gifting with partner integrations
3. Streaks and engagement features
4. Enhanced accessibility features
5. Web viewer for family members
6. AI-powered message personalization

---

**Estimated Timeline**: 1-2 weeks for MVP with dedicated development
**Primary Platform**: iOS (Android compatible)
**Key Dependencies**: Supabase, Expo, iOS TestFlight access

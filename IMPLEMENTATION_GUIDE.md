# Colors App - Complete Implementation Guide

## Quick Start for Agents

This is a comprehensive, step-by-step guide for implementing the Colors feelings-sharing app. Each task is designed to be executed independently by AI agents.

### Prerequisites
- Node.js 18+ installed
- Expo CLI installed globally: `npm install -g @expo/cli`
- Supabase account and project created
- iOS Simulator or physical device for testing

## TASK 1: Project Initialization

### Execute these commands:
```bash
cd /home/ubuntu/repos/Colors
npx create-expo-app@latest . --template blank-typescript
npx expo install @supabase/supabase-js expo-contacts expo-sms expo-notifications react-native-svg expo-router zod
npm install @types/react @types/react-native
```

### Create project structure:
```bash
mkdir -p app/{auth,tabs,groups,modals}
mkdir -p components lib supabase/{functions,sql} types
```

### Update app.json:
```json
{
  "expo": {
    "name": "Colors",
    "slug": "colors-app",
    "version": "1.0.0",
    "orientation": "portrait",
    "icon": "./assets/icon.png",
    "userInterfaceStyle": "light",
    "splash": {
      "image": "./assets/splash.png",
      "resizeMode": "contain",
      "backgroundColor": "#ffffff"
    },
    "assetBundlePatterns": ["**/*"],
    "ios": {
      "supportsTablet": true,
      "bundleIdentifier": "com.colors.app"
    },
    "android": {
      "adaptiveIcon": {
        "foregroundImage": "./assets/adaptive-icon.png",
        "backgroundColor": "#FFFFFF"
      },
      "package": "com.colors.app"
    },
    "web": {
      "favicon": "./assets/favicon.png"
    },
    "scheme": "colors",
    "plugins": [
      "expo-router",
      [
        "expo-notifications",
        {
          "icon": "./assets/notification-icon.png",
          "color": "#ffffff"
        }
      ]
    ]
  }
}
```

**Verification:** Run `npx expo start` - should start without errors

## TASK 2: Supabase Setup

### Create database schema file:
Create `supabase/sql/001_schema.sql` with complete schema (see full schema in main task plan)

### Create Supabase client:
Create `lib/supabase.ts`:
```typescript
import { createClient } from '@supabase/supabase-js'
import { Database } from '../types/database'

const supabaseUrl = process.env.EXPO_PUBLIC_SUPABASE_URL!
const supabaseAnonKey = process.env.EXPO_PUBLIC_SUPABASE_ANON_KEY!

export const supabase = createClient<Database>(supabaseUrl, supabaseAnonKey)
```

### Create environment file:
Create `.env`:
```
EXPO_PUBLIC_SUPABASE_URL=your_supabase_url
EXPO_PUBLIC_SUPABASE_ANON_KEY=your_anon_key
EXPO_PUBLIC_APP_SCHEME=colors
```

**Verification:** Supabase client connects without errors

## TASK 3: Authentication System

### Create phone auth screen:
Create `app/(auth)/phone.tsx`:
```typescript
import React, { useState } from 'react'
import { View, Text, TextInput, TouchableOpacity, Alert } from 'react-native'
import { supabase } from '../../lib/supabase'
import { router } from 'expo-router'

export default function PhoneScreen() {
  const [phone, setPhone] = useState('')
  const [loading, setLoading] = useState(false)

  const sendOTP = async () => {
    setLoading(true)
    const { error } = await supabase.auth.signInWithOtp({
      phone: phone,
    })
    
    if (error) {
      Alert.alert('Error', error.message)
    } else {
      router.push('/(auth)/otp')
    }
    setLoading(false)
  }

  return (
    <View style={{ flex: 1, padding: 20, justifyContent: 'center' }}>
      <Text style={{ fontSize: 24, marginBottom: 20 }}>Enter your phone number</Text>
      <TextInput
        value={phone}
        onChangeText={setPhone}
        placeholder="+1234567890"
        keyboardType="phone-pad"
        style={{ borderWidth: 1, padding: 15, marginBottom: 20, borderRadius: 8 }}
      />
      <TouchableOpacity
        onPress={sendOTP}
        disabled={loading}
        style={{ backgroundColor: '#007AFF', padding: 15, borderRadius: 8 }}
      >
        <Text style={{ color: 'white', textAlign: 'center' }}>
          {loading ? 'Sending...' : 'Send Code'}
        </Text>
      </TouchableOpacity>
    </View>
  )
}
```

### Create OTP verification screen:
Create `app/(auth)/otp.tsx` with OTP input and verification logic

**Verification:** User can sign up and receive OTP

## TASK 4: Core Data Models

### Create TypeScript types:
Create `types/database.ts`:
```typescript
export interface Database {
  public: {
    Tables: {
      users: {
        Row: {
          id: string
          phone: string
          display_name: string
          avatar_url: string | null
          created_at: string
        }
        Insert: {
          id?: string
          phone: string
          display_name: string
          avatar_url?: string | null
          created_at?: string
        }
        Update: {
          id?: string
          phone?: string
          display_name?: string
          avatar_url?: string | null
          created_at?: string
        }
      }
      groups: {
        Row: {
          id: string
          owner_id: string
          name: string
          emoji: string | null
          created_at: string
        }
        Insert: {
          id?: string
          owner_id: string
          name: string
          emoji?: string | null
          created_at?: string
        }
        Update: {
          id?: string
          owner_id?: string
          name?: string
          emoji?: string | null
          created_at?: string
        }
      }
      feelings: {
        Row: {
          id: string
          group_id: string
          user_id: string
          day: string
          color: string
          word: string
          why: string | null
          created_at: string
        }
        Insert: {
          id?: string
          group_id: string
          user_id: string
          day: string
          color: string
          word: string
          why?: string | null
          created_at?: string
        }
        Update: {
          id?: string
          group_id?: string
          user_id?: string
          day?: string
          color?: string
          word?: string
          why?: string | null
          created_at?: string
        }
      }
    }
  }
}
```

**Verification:** TypeScript compilation succeeds with proper types

## TASK 5: Color & Word System

### Create color mappings:
Create `lib/colorMap.ts`:
```typescript
export const COLORS = {
  BLUE: '#3B82F6',
  GREEN: '#10B981', 
  YELLOW: '#F59E0B',
  ORANGE: '#F97316',
  RED: '#EF4444',
  PURPLE: '#8B5CF6',
  TEAL: '#14B8A6',
  GRAY: '#6B7280'
} as const

export const COLOR_WORDS = {
  BLUE: ['disappointed', 'sad', 'down', 'melancholy'],
  GREEN: ['calm', 'content', 'grateful', 'peaceful'],
  YELLOW: ['happy', 'excited', 'upbeat', 'joyful'],
  ORANGE: ['energetic', 'motivated', 'eager', 'enthusiastic'],
  RED: ['angry', 'frustrated', 'stressed', 'irritated'],
  PURPLE: ['reflective', 'thoughtful', 'nostalgic', 'contemplative'],
  TEAL: ['curious', 'hopeful', 'optimistic', 'inspired'],
  GRAY: ['tired', 'meh', 'neutral', 'indifferent']
} as const

export type ColorName = keyof typeof COLORS
export type ColorValue = typeof COLORS[ColorName]
```

### Create color picker component:
Create `components/ColorPicker.tsx` with grid of color options

### Create word picker component:
Create `components/WordPicker.tsx` with word chips based on selected color

**Verification:** Color and word selection works smoothly

## TASK 6: Mood Ring Visualization

### Create SVG ring component:
Create `components/MoodRing.tsx`:
```typescript
import React from 'react'
import { View, TouchableOpacity } from 'react-native'
import Svg, { Path } from 'react-native-svg'
import { COLORS } from '../lib/colorMap'

interface MoodRingProps {
  members: Array<{
    id: string
    name: string
    color?: string
    isStale?: boolean
  }>
  onSliceTap: (memberId: string) => void
}

export default function MoodRing({ members, onSliceTap }: MoodRingProps) {
  const radius = 100
  const centerX = 120
  const centerY = 120
  const sliceAngle = 60 // 360/6 degrees per slice

  const createSlicePath = (index: number) => {
    const startAngle = index * sliceAngle - 90 // Start from top
    const endAngle = startAngle + sliceAngle
    
    const startAngleRad = (startAngle * Math.PI) / 180
    const endAngleRad = (endAngle * Math.PI) / 180
    
    const x1 = centerX + radius * Math.cos(startAngleRad)
    const y1 = centerY + radius * Math.sin(startAngleRad)
    const x2 = centerX + radius * Math.cos(endAngleRad)
    const y2 = centerY + radius * Math.sin(endAngleRad)
    
    return `M ${centerX} ${centerY} L ${x1} ${y1} A ${radius} ${radius} 0 0 1 ${x2} ${y2} Z`
  }

  return (
    <View style={{ alignItems: 'center', justifyContent: 'center' }}>
      <Svg width={240} height={240}>
        {Array.from({ length: 6 }).map((_, index) => {
          const member = members[index]
          const color = member?.color ? COLORS[member.color as keyof typeof COLORS] : '#E5E7EB'
          const opacity = member?.isStale ? 0.4 : 1
          
          return (
            <TouchableOpacity key={index} onPress={() => member && onSliceTap(member.id)}>
              <Path
                d={createSlicePath(index)}
                fill={color}
                opacity={opacity}
                stroke="#FFFFFF"
                strokeWidth={2}
              />
            </TouchableOpacity>
          )
        })}
      </Svg>
    </View>
  )
}
```

**Verification:** Ring displays correctly with 6 slices, colors update based on data

## TASK 7: Group Management

### Create group creation flow:
Create `app/modals/createGroup.tsx` with:
- Group name input
- Emoji picker
- Contact selection
- Invite sending via SMS

### Create group list screen:
Create `app/(tabs)/home.tsx` showing user's groups

### Create group detail screen:
Create `app/groups/[id].tsx` with mood ring and member management

**Verification:** Users can create groups, invite friends, and view group rings

## TASK 8: Feelings Logging

### Create feeling log modal:
Create `app/modals/logFeeling.tsx`:
```typescript
import React, { useState } from 'react'
import { View, Text, TextInput, TouchableOpacity, Alert } from 'react-native'
import { supabase } from '../../lib/supabase'
import ColorPicker from '../../components/ColorPicker'
import WordPicker from '../../components/WordPicker'
import { ColorName } from '../../lib/colorMap'

interface LogFeelingModalProps {
  groupId: string
  onClose: () => void
}

export default function LogFeelingModal({ groupId, onClose }: LogFeelingModalProps) {
  const [selectedColor, setSelectedColor] = useState<ColorName | null>(null)
  const [selectedWord, setSelectedWord] = useState('')
  const [why, setWhy] = useState('')
  const [loading, setLoading] = useState(false)

  const saveFeling = async () => {
    if (!selectedColor || !selectedWord) {
      Alert.alert('Error', 'Please select a color and word')
      return
    }

    setLoading(true)
    const today = new Date().toISOString().split('T')[0]
    
    const { error } = await supabase
      .from('feelings')
      .insert({
        group_id: groupId,
        user_id: (await supabase.auth.getUser()).data.user?.id!,
        day: today,
        color: selectedColor,
        word: selectedWord,
        why: why || null
      })

    if (error) {
      Alert.alert('Error', error.message)
    } else {
      onClose()
    }
    setLoading(false)
  }

  return (
    <View style={{ flex: 1, padding: 20 }}>
      <Text style={{ fontSize: 24, marginBottom: 20 }}>How are you feeling?</Text>
      
      <ColorPicker
        selectedColor={selectedColor}
        onColorSelect={setSelectedColor}
      />
      
      {selectedColor && (
        <WordPicker
          color={selectedColor}
          selectedWord={selectedWord}
          onWordSelect={setSelectedWord}
        />
      )}
      
      <TextInput
        value={why}
        onChangeText={setWhy}
        placeholder="Why? (optional)"
        multiline
        maxLength={140}
        style={{ borderWidth: 1, padding: 15, marginVertical: 20, borderRadius: 8, height: 80 }}
      />
      
      <TouchableOpacity
        onPress={saveFeling}
        disabled={loading || !selectedColor || !selectedWord}
        style={{ backgroundColor: '#007AFF', padding: 15, borderRadius: 8 }}
      >
        <Text style={{ color: 'white', textAlign: 'center' }}>
          {loading ? 'Saving...' : 'Save Feeling'}
        </Text>
      </TouchableOpacity>
    </View>
  )
}
```

**Verification:** Users can log daily feelings with color, word, and optional explanation

## TASK 9: Support Messaging

### Create message templates:
Create `lib/templates.ts`:
```typescript
export function generateSupportMessage(name: string, color: string, word: string): string {
  const w = word.toLowerCase()
  
  if (['disappointed', 'sad', 'down'].includes(w)) {
    return `yo ${name}, saw you're feeling ${word}. keep your head up — here to talk anytime.`
  }
  
  if (['stressed', 'frustrated', 'angry'].includes(w)) {
    return `hey ${name}, looks like today's heavy. want to vent or grab a quick call?`
  }
  
  if (['tired', 'meh', 'neutral'].includes(w)) {
    return `${name}, sending a little boost. if you need a break or coffee buddy, I'm in.`
  }
  
  if (['happy', 'excited', 'upbeat'].includes(w)) {
    return `love to see the ${word}! got a win to celebrate?`
  }
  
  return `hey ${name}, saw you're feeling ${word} (${color}). here if you need me.`
}
```

### Create slice detail modal:
Create `components/SliceSheet.tsx` with member details and "Send support" button

### Integrate SMS sending:
```typescript
import * as SMS from 'expo-sms'

const sendSupportMessage = async (phone: string, message: string) => {
  const isAvailable = await SMS.isAvailableAsync()
  if (isAvailable) {
    await SMS.sendSMSAsync([phone], message)
  }
}
```

**Verification:** Users can tap slices and send prefilled support messages

## TASK 10: Push Notifications

### Set up notification permissions:
Create `lib/notifications.ts`:
```typescript
import * as Notifications from 'expo-notifications'
import { supabase } from './supabase'

export async function registerForPushNotifications() {
  const { status: existingStatus } = await Notifications.getPermissionsAsync()
  let finalStatus = existingStatus
  
  if (existingStatus !== 'granted') {
    const { status } = await Notifications.requestPermissionsAsync()
    finalStatus = status
  }
  
  if (finalStatus !== 'granted') {
    return null
  }
  
  const token = (await Notifications.getExpoPushTokenAsync()).data
  
  // Save token to database
  const user = await supabase.auth.getUser()
  if (user.data.user) {
    await supabase
      .from('user_devices')
      .upsert({
        user_id: user.data.user.id,
        expo_push_token: token
      })
  }
  
  return token
}
```

### Create Edge function for sending pushes:
Create `supabase/functions/send_daily_pushes/index.ts` with logic to send notifications

**Verification:** Users receive random daily notifications

## TASK 11: Testing & Polish

### Create comprehensive test scenarios:
1. Complete user journey: signup → create group → invite → log feeling → view ring → send support
2. Test with 6 users in a group
3. Verify all ring states: all logged, some stale, none logged
4. Test push notifications
5. Test edge cases and error scenarios

### Add loading states and error handling:
- Loading spinners for async operations
- Error messages for failed operations
- Empty states for no data
- Offline handling

**Verification:** App works smoothly across all scenarios

## TASK 12: Deployment Preparation

### Configure app for production:
- Update app.json with production settings
- Add app icons and splash screens
- Configure bundle identifiers
- Set up signing certificates

### Create build:
```bash
npx expo build:ios
npx expo build:android
```

### Deploy to TestFlight/Play Store:
- Submit iOS build to TestFlight
- Submit Android build to Play Store internal testing

**Verification:** App successfully deploys and is accessible to testers

## Success Criteria Checklist

- [ ] User can sign up with phone OTP
- [ ] User can create a group with name and emoji  
- [ ] User can invite friends via SMS with deep links
- [ ] User can log daily feeling (color + word + optional why)
- [ ] User can see group ring with all members' latest colors
- [ ] User can tap a slice to see friend's details
- [ ] User can send prefilled supportive message via SMS
- [ ] User receives random daily push notification
- [ ] Data is private to group members only
- [ ] User can delete their account and data
- [ ] App works on both iOS and Android
- [ ] Real-time updates work correctly
- [ ] All edge cases handled gracefully

## Environment Variables Required

```
EXPO_PUBLIC_SUPABASE_URL=your_supabase_project_url
EXPO_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key (for Edge functions)
EXPO_PUBLIC_APP_SCHEME=colors
```

## Deployment Commands

```bash
# Start development server
npx expo start

# Build for iOS
npx expo build:ios

# Build for Android  
npx expo build:android

# Deploy Edge functions
supabase functions deploy generate_daily_schedule
supabase functions deploy send_daily_pushes
supabase functions deploy create_invite
supabase functions deploy message_template
```

This guide provides everything needed to build the complete Colors MVP. Each task can be executed independently by AI agents following the step-by-step instructions.

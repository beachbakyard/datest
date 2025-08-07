# Beach Volleyball Scheduling App - Claude Code Instructions

## Project Overview
Build a web application for scheduling private beach volleyball lessons with instructors, featuring booking management, payment processing, and user authentication.

## Tech Stack (Free with Scaling Options)

### Frontend
- **Next.js 14** with TypeScript
- **Tailwind CSS** for styling
- **Shadcn/ui** for UI components
- **React Hook Form** with Zod validation

### Backend & Database
- **Supabase** (free tier: 50MB database, 500MB bandwidth)
  - PostgreSQL database
  - Authentication
  - Real-time subscriptions
  - Row Level Security (RLS)

### Payment Processing
- **Stripe** (2.9% + 30¢ per transaction, no monthly fees)

### Deployment
- **Vercel** (free tier with hobby plan limits)

### Additional Services
- **Resend** for emails (free: 3,000/month, $20/month for 50k)
- **Uploadthing** for file uploads (free: 2GB storage)

## Claude Code Setup Instructions

### 1. Initial Project Setup
```bash
# Tell Claude Code to run these commands:
npx create-next-app@latest volleyball-scheduler --typescript --tailwind --eslint --app
cd volleyball-scheduler
npm install @supabase/supabase-js @supabase/auth-helpers-nextjs
npm install @radix-ui/react-slot @radix-ui/react-dialog @radix-ui/react-calendar
npm install react-hook-form @hookform/resolvers zod
npm install stripe @stripe/stripe-js
npm install lucide-react date-fns
npm install resend
npm install uploadthing @uploadthing/react
npx shadcn-ui@latest init
```

### 2. Environment Variables Setup
Create `.env.local` with:
```
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key
STRIPE_SECRET_KEY=your_stripe_secret_key
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=your_stripe_publishable_key
STRIPE_WEBHOOK_SECRET=your_webhook_secret
RESEND_API_KEY=your_resend_key
UPLOADTHING_SECRET=your_uploadthing_secret
UPLOADTHING_APP_ID=your_uploadthing_app_id
```

### 3. Database Schema (Supabase)

Ask Claude Code to create a Supabase migration file with this schema:

```sql
-- Users table (extends Supabase auth.users)
CREATE TABLE profiles (
  id UUID REFERENCES auth.users ON DELETE CASCADE,
  email TEXT UNIQUE NOT NULL,
  full_name TEXT,
  phone TEXT,
  role TEXT DEFAULT 'student' CHECK (role IN ('student', 'instructor', 'admin')),
  avatar_url TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  PRIMARY KEY (id)
);

-- Instructors table
CREATE TABLE instructors (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES profiles(id) ON DELETE CASCADE,
  bio TEXT,
  experience_years INTEGER,
  hourly_rate DECIMAL(10,2) NOT NULL,
  specialties TEXT[],
  availability JSONB, -- Store weekly availability
  is_active BOOLEAN DEFAULT true,
  stripe_account_id TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Locations table
CREATE TABLE locations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  address TEXT NOT NULL,
  latitude DECIMAL(10,8),
  longitude DECIMAL(11,8),
  description TEXT,
  amenities TEXT[],
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Lessons table
CREATE TABLE lessons (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  instructor_id UUID REFERENCES instructors(id),
  student_id UUID REFERENCES profiles(id),
  location_id UUID REFERENCES locations(id),
  start_time TIMESTAMP WITH TIME ZONE NOT NULL,
  end_time TIMESTAMP WITH TIME ZONE NOT NULL,
  status TEXT DEFAULT 'scheduled' CHECK (status IN ('scheduled', 'completed', 'cancelled', 'no_show')),
  price DECIMAL(10,2) NOT NULL,
  payment_status TEXT DEFAULT 'pending' CHECK (payment_status IN ('pending', 'paid', 'refunded')),
  stripe_payment_intent_id TEXT,
  notes TEXT,
  student_notes TEXT,
  instructor_notes TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Reviews table
CREATE TABLE reviews (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  lesson_id UUID REFERENCES lessons(id),
  reviewer_id UUID REFERENCES profiles(id),
  reviewee_id UUID REFERENCES profiles(id),
  rating INTEGER CHECK (rating >= 1 AND rating <= 5),
  comment TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Enable RLS
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE instructors ENABLE ROW LEVEL SECURITY;
ALTER TABLE locations ENABLE ROW LEVEL SECURITY;
ALTER TABLE lessons ENABLE ROW LEVEL SECURITY;
ALTER TABLE reviews ENABLE ROW LEVEL SECURITY;

-- RLS Policies
CREATE POLICY "Users can view own profile" ON profiles FOR SELECT USING (auth.uid() = id);
CREATE POLICY "Users can update own profile" ON profiles FOR UPDATE USING (auth.uid() = id);

CREATE POLICY "Anyone can view active instructors" ON instructors FOR SELECT USING (is_active = true);
CREATE POLICY "Instructors can update own profile" ON instructors FOR UPDATE USING (user_id = auth.uid());

CREATE POLICY "Anyone can view active locations" ON locations FOR SELECT USING (is_active = true);

CREATE POLICY "Users can view own lessons" ON lessons FOR SELECT USING (
  student_id = auth.uid() OR 
  instructor_id IN (SELECT id FROM instructors WHERE user_id = auth.uid())
);
```

### 4. Key Components to Build

Ask Claude Code to create these components:

#### Authentication Components
- `components/auth/LoginForm.tsx`
- `components/auth/SignUpForm.tsx`
- `components/auth/ProtectedRoute.tsx`

#### Booking Components
- `components/booking/InstructorCard.tsx`
- `components/booking/CalendarView.tsx`
- `components/booking/TimeSlotPicker.tsx`
- `components/booking/BookingForm.tsx`
- `components/booking/PaymentForm.tsx`

#### Dashboard Components
- `components/dashboard/StudentDashboard.tsx`
- `components/dashboard/InstructorDashboard.tsx`
- `components/dashboard/LessonCard.tsx`
- `components/dashboard/ScheduleCalendar.tsx`

#### Shared Components
- `components/ui/` (using shadcn/ui components)
- `components/layout/Navigation.tsx`
- `components/layout/Footer.tsx`

### 5. API Routes Structure

Request Claude Code to create these API routes:

```
app/api/
├── auth/
│   ├── callback/route.ts
│   └── signout/route.ts
├── instructors/
│   ├── route.ts
│   └── [id]/route.ts
├── lessons/
│   ├── route.ts
│   ├── [id]/route.ts
│   └── availability/route.ts
├── payments/
│   ├── create-payment-intent/route.ts
│   └── webhooks/route.ts
├── locations/
│   └── route.ts
└── reviews/
    └── route.ts
```

### 6. Core Features to Implement

#### Phase 1: MVP Features
1. **User Authentication**
   - Email/password signup/login
   - Profile management
   - Role-based access (student/instructor)

2. **Instructor Management**
   - Instructor profiles with bio, rates, specialties
   - Availability settings (weekly schedule)
   - Photo upload capability

3. **Booking System**
   - Browse available instructors
   - View instructor availability
   - Book lessons with date/time selection
   - Location selection

4. **Payment Integration**
   - Stripe payment processing
   - Secure checkout flow
   - Payment confirmation

5. **Basic Dashboard**
   - Upcoming lessons view
   - Booking history
   - Basic lesson management

#### Phase 2: Enhanced Features
1. **Advanced Scheduling**
   - Recurring lesson booking
   - Cancellation/rescheduling policies
   - Automated reminders

2. **Review System**
   - Post-lesson reviews and ratings
   - Instructor rating aggregation

3. **Enhanced Communication**
   - In-app messaging
   - Email notifications
   - SMS reminders (via Twilio)

### 7. Deployment Instructions

1. **Supabase Setup**
   - Create project at supabase.com
   - Run database migrations
   - Configure authentication settings
   - Set up RLS policies

2. **Stripe Setup**
   - Create Stripe account
   - Configure webhooks
   - Set up connected accounts for instructors

3. **Vercel Deployment**
   - Connect GitHub repository
   - Add environment variables
   - Deploy with automatic builds

### 8. Scaling Considerations

#### Free Tier Limits & Upgrades:
- **Supabase**: Free (50MB DB) → Pro ($25/month, 8GB DB)
- **Vercel**: Free → Pro ($20/month, better performance)
- **Stripe**: Pay per transaction (2.9% + 30¢)
- **Resend**: Free (3k emails) → Pro ($20/month, 50k emails)

#### Performance Optimizations:
- Implement database indexing
- Add Redis caching (Upstash free tier)
- Optimize images with Next.js Image component
- Implement lazy loading

### 9. Claude Code Development Workflow

1. Start with database schema and Supabase setup
2. Build authentication system first
3. Create instructor profile system
4. Implement booking flow
5. Add payment integration
6. Build dashboards
7. Add review system
8. Implement notifications
9. Deploy and test
10. Iterate based on user feedback

### 10. Key Commands for Claude Code

```bash
# Database operations
npm run db:generate  # Generate types from Supabase
npm run db:push      # Push schema changes

# Development
npm run dev          # Start development server
npm run build        # Build for production
npm run type-check   # TypeScript validation

# Testing
npm run test         # Run tests
npm run test:e2e     # End-to-end tests
```

### 11. Security Considerations

- Implement proper RLS policies in Supabase
- Validate all inputs with Zod schemas
- Secure API routes with authentication checks
- Use HTTPS everywhere
- Implement rate limiting
- Sanitize user inputs
- Follow OWASP security guidelines

This structure provides a solid foundation for a scalable beach volleyball scheduling app that can grow from a free MVP to a full-featured paid application.
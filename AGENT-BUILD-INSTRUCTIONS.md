# Agent Instructions: Building AI Retail Image Analysis Platform

## 🎯 Mission Statement
Build a comprehensive AI-powered retail image analysis platform that processes images in batches, provides advanced filtering, and delivers detailed analytics. Follow this exact specification to create a production-ready application.

## 📋 Prerequisites Checklist
- [ ] Next.js 15+ development environment
- [ ] Supabase project with PostgreSQL database
- [ ] Google Gemini API key
- [ ] TypeScript and Tailwind CSS knowledge
- [ ] Understanding of React hooks and state management

## 🏗️ Phase 1: Project Setup & Foundation

### Step 1.1: Initialize Next.js Project
```bash
npx create-next-app@latest ai-retail-app --typescript --tailwind --eslint --app
cd ai-retail-app
```

### Step 1.2: Install Required Dependencies
```bash
npm install @supabase/supabase-js @google/generative-ai lucide-react xlsx
npm install -D @types/node
```

### Step 1.3: Environment Configuration
Create `.env.local`:
```env
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
GOOGLE_GEMINI_API_KEY=your_gemini_api_key
```

### Step 1.4: Database Schema Setup
Execute in Supabase SQL Editor:
```sql
-- Images table (primary data store)
CREATE TABLE images (
  id SERIAL PRIMARY KEY,
  project_name VARCHAR NOT NULL,
  probe_id BIGINT NOT NULL UNIQUE,
  image_taken_time TIMESTAMPTZ NOT NULL,
  scene_id BIGINT NOT NULL,
  visit_id UUID NOT NULL,
  probe_status VARCHAR NOT NULL,
  template_id INTEGER NOT NULL,
  template_name VARCHAR NOT NULL,
  store_name VARCHAR NOT NULL,
  creation_time TIMESTAMP NOT NULL,
  validation_time TIMESTAMP,
  match_count INTEGER,
  original_width INTEGER NOT NULL,
  original_height INTEGER NOT NULL,
  probe_image_path TEXT NOT NULL,
  has_flaws VARCHAR DEFAULT 'No',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Instructions table (AI prompts)
CREATE TABLE instructions (
  id SERIAL PRIMARY KEY,
  name VARCHAR NOT NULL,
  details TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Analysis results table
CREATE TABLE image_analysis_results (
  id SERIAL PRIMARY KEY,
  image_id INTEGER REFERENCES images(id) ON DELETE CASCADE,
  instruction_id INTEGER REFERENCES instructions(id) ON DELETE CASCADE,
  result_json JSONB NOT NULL,
  confidence_score NUMERIC,
  analyzed_at TIMESTAMPTZ DEFAULT NOW()
);

-- Insert sample instructions
INSERT INTO instructions (name, details) VALUES 
('Equipment Detection', 'You are quality assurance specialist. Step 1: You should classify equipment into ONLY these 2 types: 1. Endcap - usually one brand per door / bay, usually no price tag below product and one big price tag below endcap 2. Regular door - usually many brands per door / bay, usually price tag below each product. Return results in JSON format: { "equipment_type": { "type": "endcap", "confidence": 0.92 } }'),
('Brand Recognition Focus', 'Analyze this retail image and identify all visible product brands. Focus on brand names, logos, and product packaging. Return results in JSON format with brands array containing name and confidence for each detected brand.');
```

## 🏗️ Phase 2: Core Infrastructure

### Step 2.1: Create Supabase Client (`lib/supabase.ts`)
```typescript
import { createClient } from '@supabase/supabase-js'

const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL!
const supabaseAnonKey = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!

export const supabase = createClient(supabaseUrl, supabaseAnonKey)

// Type definitions
export interface Image {
  id: number
  project_name: string
  probe_id: number
  image_taken_time: string
  scene_id: number
  visit_id: string
  probe_status: string
  template_id: number
  template_name: string
  store_name: string
  creation_time: string
  validation_time?: string
  match_count?: number
  original_width: number
  original_height: number
  probe_image_path: string
  has_flaws?: string
  created_at?: string
  updated_at?: string
}

export interface Instruction {
  id: number
  name: string
  details: string
  created_at?: string
  updated_at?: string
}

export interface ImageAnalysisResult {
  id: number
  image_id: number
  instruction_id: number
  result_json: Record<string, unknown>
  confidence_score?: number
  analyzed_at?: string
}
```

### Step 2.2: Create Gemini AI Client (`lib/gemini.ts`)
```typescript
import { GoogleGenerativeAI } from '@google/generative-ai'

const genAI = new GoogleGenerativeAI(process.env.GOOGLE_GEMINI_API_KEY!)

export interface AnalysisResult {
  equipment_type?: {
    type: string
    confidence: number
  }
  products?: Array<{
    name: string
    brand?: string
    category: string
    confidence: number
    location?: { x: number; y: number; width: number; height: number }
  }>
  equipment?: Array<{
    type: string
    condition: string
    confidence: number
  }>
  brands?: Array<{
    name: string
    confidence: number
  }>
  summary?: string
  timestamp: string
  [key: string]: any // Flexible structure support
}

// Rate limiting configuration
const RATE_LIMITS = {
  FREE_TIER: {
    maxConcurrent: 10,
    delayBetweenBatches: 60000, // 1 minute
    maxPerMinute: 10
  },
  PAID_TIER: {
    maxConcurrent: 50,
    delayBetweenBatches: 6000, // 6 seconds
    maxPerMinute: 1000
  }
}

export async function analyzeImageWithGemini(
  imageUrl: string,
  instruction: string
): Promise<AnalysisResult> {
  const model = genAI.getGenerativeModel({ model: "gemini-2.5-flash" })
  
  try {
    const result = await model.generateContent([
      instruction,
      {
        inlineData: {
          mimeType: "image/jpeg",
          data: await fetchImageAsBase64(imageUrl)
        }
      }
    ])
    
    const response = await result.response
    const text = response.text()
    
    // Clean and parse JSON response
    const cleanedText = text.replace(/```json\n?|\n?```/g, '').trim()
    const analysisResult = JSON.parse(cleanedText)
    
    return {
      ...analysisResult,
      timestamp: new Date().toISOString()
    }
  } catch (error) {
    console.error('Gemini API error:', error)
    throw new Error(`Analysis failed: ${error instanceof Error ? error.message : 'Unknown error'}`)
  }
}

// Parallel processing function
export async function batchAnalyzeImagesParallel(
  imageUrls: string[],
  instruction: string,
  onProgress?: (completed: number, total: number, currentImages?: string[]) => void
): Promise<Array<{ imageUrl: string; result: AnalysisResult; error?: string }>> {
  const tier = RATE_LIMITS.FREE_TIER // Adjust based on your tier
  const results: Array<{ imageUrl: string; result: AnalysisResult; error?: string }> = []
  let completed = 0

  for (let i = 0; i < imageUrls.length; i += tier.maxConcurrent) {
    const batch = imageUrls.slice(i, i + tier.maxConcurrent)
    const currentImages = batch.map(url => url.split('/').pop() || url)
    
    if (onProgress) {
      onProgress(completed, imageUrls.length, currentImages)
    }

    // Process batch in parallel
    const batchPromises = batch.map(async (imageUrl) => {
      try {
        const result = await analyzeImageWithGemini(imageUrl, instruction)
        return { imageUrl, result }
      } catch (error) {
        return {
          imageUrl,
          result: {
            summary: `Error: ${error instanceof Error ? error.message : 'Unknown error'}`,
            timestamp: new Date().toISOString()
          },
          error: error instanceof Error ? error.message : 'Unknown error'
        }
      }
    })

    const batchResults = await Promise.all(batchPromises)
    results.push(...batchResults)
    completed += batch.length

    if (onProgress) {
      onProgress(completed, imageUrls.length)
    }

    // Rate limiting delay between batches
    if (i + tier.maxConcurrent < imageUrls.length) {
      await new Promise(resolve => setTimeout(resolve, tier.delayBetweenBatches))
    }
  }

  return results
}

async function fetchImageAsBase64(imageUrl: string): Promise<string> {
  const response = await fetch(imageUrl)
  const arrayBuffer = await response.arrayBuffer()
  const base64 = Buffer.from(arrayBuffer).toString('base64')
  return base64
}
```

## 🏗️ Phase 3: Navigation & Layout

### Step 3.1: Create Navigation Component (`components/Navigation.tsx`)
```typescript
'use client'

import { useState } from 'react'
import Link from 'next/link'
import { usePathname } from 'next/navigation'
import { Menu, X, BarChart3, Search, Zap, Settings } from 'lucide-react'

export default function Navigation() {
  const [isOpen, setIsOpen] = useState(false)
  const pathname = usePathname()

  const navigation = [
    { name: 'Dashboard', href: '/results', icon: BarChart3 },
    { name: 'Instructions', href: '/instructions', icon: Settings },
    { name: 'Analysis', href: '/analysis', icon: Zap },
  ]

  return (
    <>
      {/* Mobile menu button */}
      <div className="lg:hidden fixed top-4 left-4 z-50">
        <button
          onClick={() => setIsOpen(!isOpen)}
          className="p-2 rounded-md bg-white shadow-lg border"
        >
          {isOpen ? <X className="h-6 w-6" /> : <Menu className="h-6 w-6" />}
        </button>
      </div>

      {/* Sidebar */}
      <div className={`fixed inset-y-0 left-0 z-40 w-64 bg-white border-r transform transition-transform duration-300 ease-in-out ${
        isOpen ? 'translate-x-0' : '-translate-x-full'
      } lg:translate-x-0`}>
        
        {/* Header */}
        <div className="p-6 border-b">
          <h1 className="text-xl font-bold text-gray-900">AI Retail Analysis</h1>
          <p className="text-sm text-gray-600 mt-1">Image Processing Platform</p>
        </div>

        {/* Navigation */}
        <nav className="p-4 space-y-2">
          {navigation.map((item) => {
            const Icon = item.icon
            const isActive = pathname === item.href
            
            return (
              <Link
                key={item.name}
                href={item.href}
                className={`flex items-center px-4 py-3 rounded-lg transition-colors ${
                  isActive
                    ? 'bg-blue-50 text-blue-700 border-r-2 border-blue-700'
                    : 'text-gray-700 hover:bg-gray-50'
                }`}
                onClick={() => setIsOpen(false)}
              >
                <Icon className="h-5 w-5 mr-3" />
                {item.name}
              </Link>
            )
          })}
        </nav>

        {/* Footer */}
        <div className="absolute bottom-0 left-0 right-0 p-4 border-t bg-gray-50">
          <p className="text-xs text-gray-500 text-center">
            Powered by Gemini AI
          </p>
        </div>
      </div>

      {/* Mobile overlay */}
      {isOpen && (
        <div
          className="fixed inset-0 z-30 bg-black bg-opacity-50 lg:hidden"
          onClick={() => setIsOpen(false)}
        />
      )}
    </>
  )
}
```

### Step 3.2: Update Root Layout (`app/layout.tsx`)
```typescript
import type { Metadata } from 'next'
import { Inter } from 'next/font/google'
import './globals.css'
import Navigation from '@/components/Navigation'

const inter = Inter({ subsets: ['latin'] })

export const metadata: Metadata = {
  title: 'AI Retail Analysis Platform',
  description: 'AI-powered retail image analysis with Google Gemini',
}

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en">
      <body className={inter.className}>
        <div className="min-h-screen bg-gray-50">
          <Navigation />
          <main className="lg:ml-64 p-6">
            {children}
          </main>
        </div>
      </body>
    </html>
  )
}
```

## 🏗️ Phase 4: Dashboard Implementation

### Step 4.1: Create Dashboard Page (`app/results/page.tsx`)

**CRITICAL REQUIREMENTS:**
1. **Statistics Dashboard**: Show total stores, analyzed stores, total images, processed images
2. **Excel Upload Integration**: Direct upload with progress tracking
3. **Ultra-Minimal Image Cards**: Pinterest-style with overlay controls only
4. **Advanced Filtering**: Date, search, status, instruction, equipment type
5. **Smart Toolbar**: Icon-only buttons with expandable panels
6. **Equipment Type Filtering**: Purple-accented dropdown in filter panel
7. **Modal Image Viewer**: Detailed analysis results
8. **Export Capabilities**: JSON/CSV export

**Key Implementation Points:**
- Use 5-column filter grid: Date Start, Date End, Status, Instruction, Equipment Type
- Equipment filter uses purple accent (#8B5CF6)
- Image cards show only: image, status badge, delete button, view details button
- Filter panel expands below toolbar
- Real-time statistics updates after uploads

### Step 4.2: Dashboard Features Checklist
- [ ] Statistics cards with colored icons
- [ ] Excel upload with duplicate detection by probe_id
- [ ] Ultra-compact image grid (4-5 columns)
- [ ] Search panel (expandable)
- [ ] Filter panel with 5 filters (expandable)
- [ ] Equipment type filtering (purple accent)
- [ ] Image modal with analysis results
- [ ] Delete functionality (image vs analysis)
- [ ] Export buttons (JSON/CSV)
- [ ] Pagination controls

## 🏗️ Phase 5: Analysis Page Implementation

### Step 5.1: Create Analysis Page (`app/analysis/page.tsx`)

**CRITICAL REQUIREMENTS:**
1. **Image Selection Grid**: Paginated with checkboxes and visual feedback
2. **Smart Filtering**: Hide already analyzed images by selected instruction
3. **Parallel Processing**: 10 concurrent requests with rate limit compliance
4. **Real-time Progress**: Show current batch being processed
5. **Processing Toggle**: Switch between parallel and sequential modes
6. **Results Export**: JSON export with timestamped results

**Key Implementation Points:**
- Default to hiding analyzed images when instruction is selected
- Show purple ring borders on selected images
- Display up to 10 images being processed simultaneously
- 1-minute delays between batches for rate limit compliance
- Progress bar with current image names

### Step 5.2: Analysis Features Checklist
- [ ] Instruction selection dropdown
- [ ] Image grid with selection checkboxes
- [ ] Select All / Clear Selection buttons
- [ ] Hide analyzed images toggle (default: hide)
- [ ] Parallel processing toggle (default: enabled)
- [ ] Progress tracking with batch visualization
- [ ] Error handling for individual images
- [ ] Results export functionality
- [ ] Statistics refresh after completion

## 🏗️ Phase 6: Instructions Management

### Step 6.1: Create Instructions Page (`app/instructions/page.tsx`)

**CRITICAL REQUIREMENTS:**
1. **CRUD Operations**: Create, read, update, delete instructions
2. **Flexible JSON Support**: Handle any analysis output structure
3. **Usage Analytics**: Show which instructions are popular
4. **Example Templates**: Provide common instruction patterns

**Key Implementation Points:**
- Support for equipment_type, products, brands, custom fields
- Validation for JSON output format
- Clear examples for different analysis types
- Usage statistics integration

## 🏗️ Phase 7: Advanced Features & Polish

### Step 7.1: Error Handling & Resilience
- [ ] Comprehensive error boundaries
- [ ] API retry logic with exponential backoff
- [ ] User-friendly error messages
- [ ] Loading states for all operations
- [ ] Network failure recovery

### Step 7.2: Performance Optimization
- [ ] Image grid virtualization for large datasets
- [ ] Efficient database queries with proper indexing
- [ ] Parallel API processing with rate limit compliance
- [ ] Progressive loading and pagination
- [ ] Smart caching strategies

### Step 7.3: UI/UX Polish
- [ ] Consistent color system (blue/purple/green/red/orange)
- [ ] Smooth transitions and animations
- [ ] Responsive design for all screen sizes
- [ ] Accessibility compliance (ARIA labels, keyboard navigation)
- [ ] Professional visual design with subtle shadows

## 🚀 Phase 8: Testing & Deployment

### Step 8.1: Testing Checklist
- [ ] Excel upload with various file formats
- [ ] Parallel processing with rate limit compliance
- [ ] Equipment type filtering accuracy
- [ ] Modal viewer functionality
- [ ] Export capabilities (JSON/CSV)
- [ ] Mobile responsiveness
- [ ] Error handling scenarios

### Step 8.2: Deployment Preparation
- [ ] Environment variables configuration
- [ ] Database migrations and seeding
- [ ] API key security validation
- [ ] Performance monitoring setup
- [ ] Error logging configuration

### Step 8.3: Production Deployment
```bash
# Vercel deployment
npm run build
vercel --prod

# Environment variables in Vercel dashboard:
# NEXT_PUBLIC_SUPABASE_URL
# NEXT_PUBLIC_SUPABASE_ANON_KEY  
# GOOGLE_GEMINI_API_KEY
```

## 🎯 Success Criteria

### Functional Requirements
✅ **Excel Upload**: Parse 3000+ row files with duplicate detection  
✅ **AI Analysis**: Process 10 images simultaneously with 95%+ success rate  
✅ **Filtering**: 5-filter system with real-time results  
✅ **Export**: JSON/CSV export of filtered results  
✅ **Mobile**: Responsive design on all devices  

### Performance Requirements
✅ **Speed**: 10x faster processing with parallel mode  
✅ **Reliability**: <1% error rate on API calls  
✅ **Scalability**: Handle 10,000+ images efficiently  
✅ **UX**: <3 second page load times  

### Quality Requirements
✅ **Code**: TypeScript strict mode, comprehensive error handling  
✅ **Design**: Professional UI with consistent color system  
✅ **Security**: Secure API key management, input validation  
✅ **Documentation**: Complete specification and build instructions  

## 🚨 Critical Implementation Notes

1. **Rate Limiting**: MUST implement proper Gemini API rate limiting to avoid quota exhaustion
2. **Equipment Filtering**: MUST be on dashboard page, NOT analysis page
3. **Image Cards**: MUST be ultra-minimal with overlay controls only
4. **Parallel Processing**: MUST respect 10 RPM limit with 1-minute batch delays
5. **Color System**: MUST use consistent purple for equipment, blue for instructions
6. **Error Handling**: MUST continue batch processing on individual image failures
7. **Database Schema**: MUST match exact column names and types specified
8. **Export Format**: MUST include timestamped JSON with complete analysis results

Follow this specification exactly to build a production-ready AI retail image analysis platform that matches the reference implementation.

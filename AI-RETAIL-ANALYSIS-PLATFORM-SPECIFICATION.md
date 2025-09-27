# AI Retail Image Analysis Platform - Complete Specification

## 🎯 Project Overview

A comprehensive Next.js application for AI-powered retail image analysis using Google Gemini 2.5 Flash API and Supabase. The platform enables batch processing of retail images with custom AI instructions, advanced filtering, and detailed analytics.

## 🏗️ System Architecture

### Tech Stack
- **Frontend**: Next.js 15.5.4 with TypeScript, Tailwind CSS
- **Database**: Supabase (PostgreSQL) with real-time capabilities
- **AI Engine**: Google Gemini 2.5 Flash API
- **File Processing**: xlsx library for Excel parsing
- **Icons**: Lucide React
- **Deployment**: Vercel/Netlify compatible

### Core Database Schema

#### `images` table (Primary data store)
```sql
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
```

#### `instructions` table (AI prompts)
```sql
CREATE TABLE instructions (
  id SERIAL PRIMARY KEY,
  name VARCHAR NOT NULL,
  details TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

#### `image_analysis_results` table (AI analysis results)
```sql
CREATE TABLE image_analysis_results (
  id SERIAL PRIMARY KEY,
  image_id INTEGER REFERENCES images(id) ON DELETE CASCADE,
  instruction_id INTEGER REFERENCES instructions(id) ON DELETE CASCADE,
  result_json JSONB NOT NULL,
  confidence_score NUMERIC,
  analyzed_at TIMESTAMPTZ DEFAULT NOW()
);
```

## 🚀 Key Features & Functionality

### 1. Dashboard (Results Page) - `/results`

**Core Features:**
- **Statistics Dashboard**: Real-time metrics showing total stores, analyzed stores, total images, processed images
- **Excel Upload Integration**: Direct upload with progress tracking and duplicate detection
- **Advanced Image Grid**: Pinterest-style ultra-minimal image cards with overlay controls
- **Comprehensive Filtering System**:
  - Date range picker (start/end dates)
  - Search by store name or probe ID
  - Analysis status (all/analyzed/not analyzed)
  - Instruction-based filtering
  - **Equipment type filtering** (endcap, regular door, etc.)
- **Smart Toolbar**: Icon-only buttons (Search, Filter, Upload, Download) with expandable panels
- **Image Modal Viewer**: Detailed analysis results with tabbed interface
- **Bulk Operations**: Delete images, delete analysis results
- **Export Capabilities**: JSON and CSV export of filtered results

**UI Design Principles:**
- Ultra-compact design using 75% less vertical space
- Clean minimal cards showing only images with essential overlay buttons
- No borders, subtle shadows and background colors
- 4-5 column responsive grid for maximum image density
- Purple accent for equipment filters, blue for instructions, green for status

### 2. Analysis Page - `/analysis`

**Core Features:**
- **Batch Image Selection**: Paginated image grid with individual and bulk selection
- **Smart Filtering**: Hide already analyzed images by instruction (default behavior)
- **Parallel Processing**: Up to 10 concurrent API requests respecting Gemini rate limits
- **Real-time Progress Tracking**: Shows current batch being processed with image names
- **Processing Mode Toggle**: Switch between parallel (10x faster) and sequential processing
- **Rate Limit Compliance**: 
  - Free Tier: 10 concurrent, 1-minute delays between batches
  - Paid Tier: 50 concurrent, 6-second delays between batches
- **Results Export**: JSON export with timestamped analysis results
- **Error Handling**: Individual image error tracking with batch continuation

**Analysis Workflow:**
1. Select instruction → Auto-hide already analyzed images
2. Select images (with visual feedback - purple ring borders)
3. Choose processing mode (parallel recommended)
4. Start analysis → Real-time progress with batch visualization
5. Auto-refresh dashboard statistics upon completion

### 3. Instructions Management - `/instructions`

**Core Features:**
- **CRUD Operations**: Create, read, update, delete AI analysis prompts
- **Flexible Prompt System**: Support for any JSON output structure
- **Example Instructions**:
  - Equipment Detection: Classify as endcap/regular door with confidence
  - Brand Recognition: Identify product brands and locations
  - Custom Analysis: Any structured JSON output format
- **Usage Tracking**: See which instructions are actively used

### 4. Navigation & UX

**Design Features:**
- **Left Sidebar Navigation**: Fixed 256px width with mobile hamburger menu
- **Responsive Design**: Mobile-first with desktop optimization (1200px+)
- **Icon-based Interface**: Lucide React icons throughout
- **Color Coding System**:
  - Blue: Instructions and primary actions
  - Purple: Equipment type filters
  - Green: Status indicators and success states
  - Red: Delete actions and errors
  - Orange: Equipment type displays

## 🤖 AI Integration Specifications

### Google Gemini 2.5 Flash Configuration

**Rate Limits (Free Tier):**
- 10 requests per minute (RPM)
- 250,000 tokens per minute (TPM)
- 250 requests per day (RPD)

**Parallel Processing Implementation:**
```typescript
const RATE_LIMITS = {
  FREE_TIER: {
    maxConcurrent: 10, // Maximum limit for free tier
    delayBetweenBatches: 60000, // 1 minute between batches
    maxPerMinute: 10
  },
  PAID_TIER: {
    maxConcurrent: 50, // Conservative limit for paid tier
    delayBetweenBatches: 6000, // 6 seconds between batches
    maxPerMinute: 1000
  }
}
```

**Analysis Result Structure:**
```json
{
  "equipment_type": {
    "type": "endcap",
    "confidence": 0.92
  },
  "products": [
    {
      "name": "Product name",
      "brand": "Brand name",
      "category": "Category",
      "confidence": 0.95,
      "location": { "x": 100, "y": 150, "width": 200, "height": 300 }
    }
  ],
  "equipment": [
    {
      "type": "Equipment type",
      "condition": "good/fair/poor",
      "confidence": 0.90
    }
  ],
  "brands": [
    {
      "name": "Brand name",
      "confidence": 0.85
    }
  ],
  "summary": "Analysis summary",
  "timestamp": "2024-01-01T00:00:00.000Z"
}
```

## 📊 Data Processing Pipeline

### Excel Upload Process
1. **File Validation**: Check structure and required columns
2. **Duplicate Detection**: Compare by `probe_id` against existing records
3. **Batch Processing**: Insert in 100-record batches with progress tracking
4. **Error Handling**: Individual row error tracking with batch continuation
5. **Statistics Update**: Real-time dashboard refresh

**Required Excel Columns:**
- A: Project Name
- B: Probe Id (unique identifier)
- C: Image Taken Time
- J: Store Name
- X: Probe Image Path (S3 URLs)
- Additional metadata columns supported

### Analysis Processing Pipeline
1. **Image Selection**: Filter by instruction to avoid duplicates
2. **Batch Creation**: Group into rate-limit compliant batches
3. **Parallel Execution**: Process multiple images simultaneously
4. **Result Storage**: Store in `image_analysis_results` with confidence scores
5. **Statistics Update**: Refresh dashboard metrics

## 🎨 UI/UX Design System

### Layout Principles
- **Ultra-Compact Design**: 75% space reduction through optimized padding and sizing
- **Visual Hierarchy**: Clear information architecture with consistent spacing
- **Minimal Interface**: Focus on content with essential controls only
- **Responsive Grid**: 4-5 column image grid with proper aspect ratios

### Component Architecture
```
├── Navigation (Sidebar)
├── Dashboard (Results Page)
│   ├── Statistics Cards
│   ├── Toolbar (Search, Filter, Upload, Download)
│   ├── Image Grid
│   └── Modal Viewer
├── Analysis Page
│   ├── Control Panel
│   ├── Image Selection Grid
│   └── Progress Tracking
└── Instructions Management
    ├── Instruction List
    └── CRUD Forms
```

### Color System
- **Primary Blue**: #3B82F6 (instructions, primary actions)
- **Purple**: #8B5CF6 (equipment filters)
- **Green**: #10B981 (success, status indicators)
- **Red**: #EF4444 (delete actions, errors)
- **Orange**: #F59E0B (equipment type displays)
- **Gray Scale**: #F9FAFB to #111827 (backgrounds, text)

## 🔧 Technical Implementation Details

### Environment Configuration
```env
# Supabase Configuration
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key

# Google Gemini API
GOOGLE_GEMINI_API_KEY=your_gemini_api_key
```

### Key Dependencies
```json
{
  "dependencies": {
    "next": "15.5.4",
    "react": "^18.2.0",
    "typescript": "^5.0.0",
    "tailwindcss": "^3.3.0",
    "@supabase/supabase-js": "^2.38.0",
    "@google/generative-ai": "^0.2.1",
    "lucide-react": "^0.263.1",
    "xlsx": "^0.18.5"
  }
}
```

### Performance Optimizations
- **Parallel API Processing**: 10x faster analysis with rate limit compliance
- **Efficient Database Queries**: Optimized joins and filtering
- **Image Grid Virtualization**: Handle large datasets efficiently
- **Progressive Loading**: Paginated results with smooth navigation
- **Smart Caching**: Minimize redundant API calls

## 🚦 Error Handling & Resilience

### API Error Handling
- **Rate Limit Management**: Automatic retry with exponential backoff
- **Individual Image Errors**: Continue batch processing on single failures
- **Network Resilience**: Timeout handling and retry logic
- **User Feedback**: Clear error messages with actionable guidance

### Data Validation
- **Excel Structure Validation**: Verify required columns before processing
- **Image URL Validation**: Check accessibility before analysis
- **Input Sanitization**: Prevent injection attacks and data corruption
- **Type Safety**: Full TypeScript implementation with strict types

## 📈 Analytics & Monitoring

### Built-in Metrics
- **Processing Statistics**: Success/failure rates, processing times
- **Usage Analytics**: Most used instructions, popular equipment types
- **Performance Monitoring**: API response times, batch processing efficiency
- **Error Tracking**: Categorized error logging with resolution guidance

### Export Capabilities
- **JSON Export**: Complete analysis results with metadata
- **CSV Export**: Tabular data for spreadsheet analysis
- **Filtered Exports**: Export only selected/filtered results
- **Timestamped Results**: Audit trail for all operations

## 🔐 Security & Privacy

### Data Protection
- **Supabase RLS**: Row-level security policies
- **API Key Management**: Secure environment variable handling
- **Input Validation**: Comprehensive sanitization
- **HTTPS Enforcement**: Secure data transmission

### Access Control
- **Authentication Ready**: Supabase Auth integration prepared
- **Role-based Access**: Configurable user permissions
- **Audit Logging**: Track all data modifications
- **Data Retention**: Configurable cleanup policies

## 🚀 Deployment & Scaling

### Deployment Options
- **Vercel**: Recommended for Next.js applications
- **Netlify**: Alternative with similar capabilities
- **Docker**: Containerized deployment for custom infrastructure
- **Supabase Hosting**: Database and backend services

### Scaling Considerations
- **Database Indexing**: Optimized queries for large datasets
- **CDN Integration**: Fast image loading globally
- **API Rate Management**: Upgrade paths for higher throughput
- **Horizontal Scaling**: Stateless architecture for load balancing

---

This specification provides a complete blueprint for building an AI-powered retail image analysis platform with enterprise-grade features, performance optimization, and user experience design.

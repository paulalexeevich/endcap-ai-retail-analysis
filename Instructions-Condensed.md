# AI Image Processing App - Development Guide

## Overview
Multi-page Next.js app for retail image analysis: Excel upload → Gallery → AI Processing → Results

**Tech Stack**: Next.js 14 + TypeScript + Tailwind + Supabase + Google Gemini 2.5 Flash

## Database (Existing Supabase: ybzoioqgbvcxqiejopja)

### Tables
- `images` (1,654 rows): probe_id (unique), probe_image_path (S3), image_taken_time, store_name, etc.
- `instructions`: id, name, details (AI prompts)
- `image_analysis_results`: image_id, instruction_id, result_json (JSONB)

### Excel Mapping (26 columns A-Z)
Key columns: B=probe_id, C=image_taken_time, J=store_name, X=probe_image_path, Z=has_flaws

## Pages & Features

### 1. Upload (`/upload`)
- Excel file upload with xlsx library
- Duplicate detection by probe_id (skip duplicates)
- Progress indicator

### 2. Gallery (`/gallery`) 
- 10 images per page grid
- Filters: date range, processing status (processed/not processed), instruction used
- Image modal with metadata

### 3. Instructions (`/instructions`)
- CRUD for AI analysis prompts
- Existing: "Equipment Detection", "Brand Recognition Focus"

### 4. Analysis (`/analysis`)
- Batch image selection
- Multiple processing queues
- Real-time progress tracking
- Gemini API integration

### 5. Results (`/results`)
- View analysis results
- Export functionality

## Key Implementation

### Setup
```bash
npx create-next-app@latest --typescript --tailwind --app
npm install @supabase/supabase-js @google/generative-ai xlsx lucide-react
```

### Environment
```env
NEXT_PUBLIC_SUPABASE_URL=https://ybzoioqgbvcxqiejopja.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=[from Supabase]
GOOGLE_GEMINI_API_KEY=[from Google AI Studio]
```

### Next.js Configuration (CRITICAL)
```typescript
// next.config.ts - Required for environment variable loading
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  env: {
    GOOGLE_GEMINI_API_KEY: process.env.GOOGLE_GEMINI_API_KEY,
  },
  serverExternalPackages: ['@google/generative-ai'] // Note: moved from experimental in Next.js 15
};

export default nextConfig;
```

### Gallery Filters Query
```sql
SELECT i.*, CASE WHEN iar.id IS NOT NULL THEN 'processed' ELSE 'not_processed' END as status
FROM images i LEFT JOIN image_analysis_results iar ON i.id = iar.image_id
WHERE i.image_taken_time BETWEEN $1 AND $2
AND ($3 = 'all' OR ($3 = 'processed' AND iar.id IS NOT NULL) OR ($3 = 'not_processed' AND iar.id IS NULL))
LIMIT 10 OFFSET $4
```

### Batch Processing Flow
1. Select images + instruction
2. For each image: fetch S3 → Gemini API → parse JSON → store results
3. Real-time progress updates
4. Error handling & retries

## Development Phases
1. **Setup**: Next.js + Supabase + navigation
2. **Upload**: Excel parsing + duplicate detection
3. **Gallery**: Image grid + filtering
4. **Instructions**: CRUD interface
5. **Analysis**: Gemini integration + queues
6. **Results**: View + export
7. **Polish**: Error handling + optimization

## 🔥 CRITICAL LEARNINGS & FIXES

### Database Schema Issues
- **CRITICAL**: Column name is `analyzed_at` NOT `created_at` in `image_analysis_results`
- **Fix**: Update TypeScript interfaces and queries to use correct column names
- **Verification**: Always check actual table schema with `DESCRIBE table_name` or `information_schema.columns`

### Supabase Query Limitations
- **Issue**: Complex nested filtering on relationships fails (`image.store_name.ilike`)
- **Fix**: Simplify queries, avoid deep relationship filtering in initial load
- **Best Practice**: Load data first, then apply client-side filtering for complex cases

### Icon Import Errors
- **Issue**: `Gallery` icon doesn't exist in lucide-react
- **Fix**: Use `Images` icon instead
- **Prevention**: Verify icon names at https://lucide.dev before using

### TypeScript Strictness
- **Issue**: `any` types cause build failures
- **Fix**: Use `Record<string, unknown>` or proper interfaces
- **Best Practice**: Define proper types for all API responses and database schemas

### Environment Variable Loading ⚠️ CRITICAL
- **Issue**: Next.js 15 doesn't automatically load environment variables for server-side API calls
- **Root Cause**: Environment variables undefined in server context, causing "Invalid API key" errors
- **Fix**: Add to `next.config.ts`:
```typescript
const nextConfig: NextConfig = {
  env: {
    GOOGLE_GEMINI_API_KEY: process.env.GOOGLE_GEMINI_API_KEY,
  }
}
```
- **Additional Fix**: Server-side only logging to prevent client-side errors:
```typescript
if (!apiKey && typeof window === 'undefined') {
  console.error('Missing GOOGLE_GEMINI_API_KEY environment variable')
}
```
- **Verification**: Create `/api/test-env` endpoint to verify environment loading
- **Critical**: Restart server with explicit variable: `GOOGLE_GEMINI_API_KEY=your_key npm run dev`

### Gemini API Integration
- **Model**: Use `gemini-2.5-flash` (latest stable model with enhanced image understanding)
- **API Structure**: Use proper contents/parts structure as per official docs
- **API Key Issues**: Free tier has strict quota limits - may need paid plan
- **Model Access**: Check available models with your API key - some require billing setup
- **Rate Limiting**: Add 1-second delays between API calls
- **Error Handling**: Wrap all API calls in try-catch with specific error messages for quota/auth issues
- **JSON Parsing**: Clean response text (remove ```json markers) before parsing

### Results Page Query Structure
```typescript
// CORRECT: Simple query structure
const { data } = await supabase
  .from('image_analysis_results')
  .select(`
    *,
    image:images(*),
    instruction:instructions(*)
  `)
  .order('analyzed_at', { ascending: false })

// AVOID: Complex nested filtering
// .gte('image.image_taken_time', startDate) // This fails
```

### Image Error Handling
```typescript
// Add proper fallback for broken images
onError={(e) => {
  const target = e.target as HTMLImageElement
  target.src = 'data:image/svg+xml;base64,PHN2ZyB3aWR0aD0i...' // Base64 fallback
}}
```

### Build Optimization
- **ESLint Warnings**: Use `/* eslint-disable-next-line */` for unavoidable warnings
- **Image Optimization**: Consider Next.js `<Image />` component for production
- **Bundle Size**: Import only needed icons: `import { Upload } from 'lucide-react'`

## Testing Strategy

### End-to-End Validation
1. **Database**: Verify schema with Supabase MCP
2. **API Keys**: Test Gemini API directly with Node.js script
3. **Real Data**: Process actual retail images, verify JSON structure
4. **UI Flow**: Test complete workflow from upload to results display

### Common Debugging
```bash
# Check table schema
SELECT column_name, data_type FROM information_schema.columns WHERE table_name = 'image_analysis_results';

# Test Gemini API and environment loading
node -e "
const { GoogleGenerativeAI } = require('@google/generative-ai');
console.log('API Key loaded:', !!process.env.GOOGLE_GEMINI_API_KEY);
if (process.env.GOOGLE_GEMINI_API_KEY) {
  const genAI = new GoogleGenerativeAI(process.env.GOOGLE_GEMINI_API_KEY);
  const model = genAI.getGenerativeModel({ model: 'gemini-2.5-flash' });
  model.generateContent('Hello').then(r => console.log('✅ API working')).catch(e => console.log('❌ Error:', e.message));
} else {
  console.log('❌ GOOGLE_GEMINI_API_KEY not set');
}
"

# Test environment loading in Next.js context
curl http://localhost:3000/api/test-env

# Check server logs for environment variable issues
tail -f .next/server/app/analysis/page.js # Look for "Missing GOOGLE_GEMINI_API_KEY"
```

### API Key Troubleshooting ⚠️ UPDATED
- **Environment Loading Issues**: Most "Invalid API key" errors are actually undefined environment variables
  - **Solution**: Update `next.config.ts` with explicit env configuration
  - **Test**: Use `/api/test-env` endpoint to verify: `curl http://localhost:3000/api/test-env`
  - **Expected**: `{"hasApiKey":true,"keyPrefix":"AIzaSy...","allEnvVars":["GOOGLE_GEMINI_API_KEY"]}`
- **Next.js 15 Changes**: `experimental.serverComponentsExternalPackages` moved to `serverExternalPackages`
- **Quota Exceeded**: Free tier has very limited requests - upgrade to paid plan
- **Model Not Found**: Some models require billing setup in Google AI Studio  
- **Authentication Error**: Verify API key is valid and not restricted
- **Rate Limits**: Add delays between requests, respect quota limits
- **Server Restart**: Always restart after environment variable changes

## Critical Notes
- Use existing Supabase tables (perfect match)
- S3 URLs for images (no direct uploads)
- Skip duplicates by probe_id
- Laptop-optimized layout (1200px+)
- Multiple simultaneous processing queues
- Structured JSON output from Gemini API
- **ALWAYS verify database schema before coding**
- **Test API integrations independently before UI integration**
- **Use Supabase MCP for real credentials and testing**
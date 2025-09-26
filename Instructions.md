# AI Image Processing App - Development Instructions

## 📋 Project Overview

Build a multi-page web application for uploading, viewing, and processing retail images using AI analysis. The app allows Excel file uploads, image gallery browsing, instruction management, and batch AI processing with Google Gemini API.

## 🎯 Core Requirements

### Functional Requirements
1. **Excel Upload System**: Upload Excel files with image metadata, detect duplicates by `probe_id`, skip duplicates
2. **Image Gallery**: Paginated view (10 images per page) with filtering and search
3. **Instruction Management**: CRUD operations for AI analysis prompts
4. **Batch AI Processing**: Queue-based image analysis using Google Gemini API
5. **Results Management**: View and export analysis results

### Technical Requirements
- **Frontend**: Next.js 14 with TypeScript
- **Styling**: Tailwind CSS
- **Database**: Existing Supabase project (ID: ybzoioqgbvcxqiejopja)
- **AI Integration**: Google Gemini 2.5 Flash API
- **Image Storage**: S3 URLs (no direct uploads)
- **Layout**: Multi-page laptop-optimized design

## 🗄️ Database Schema (Existing - Reuse)

### Tables Structure
```sql
-- images table (1,654 existing rows)
CREATE TABLE images (
    id SERIAL PRIMARY KEY,
    project_name VARCHAR,
    probe_id BIGINT UNIQUE, -- Key for duplicate detection
    image_taken_time TIMESTAMPTZ,
    scene_id BIGINT,
    visit_id UUID,
    probe_status VARCHAR,
    template_id INTEGER,
    template_name VARCHAR,
    store_name VARCHAR,
    creation_time TIMESTAMP,
    validation_time TIMESTAMP,
    match_count INTEGER,
    original_width INTEGER,
    original_height INTEGER,
    probe_image_path TEXT, -- S3 URLs
    has_flaws VARCHAR DEFAULT 'No',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- instructions table (2 existing instructions)
CREATE TABLE instructions (
    id SERIAL PRIMARY KEY,
    name VARCHAR UNIQUE,
    details TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- image_analysis_results table (empty - for new results)
CREATE TABLE image_analysis_results (
    id SERIAL PRIMARY KEY,
    image_id INTEGER REFERENCES images(id),
    instruction_id INTEGER REFERENCES instructions(id),
    result_json JSONB, -- Structured Gemini API output
    confidence_score NUMERIC,
    analyzed_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Existing Instructions
1. **Equipment Detection**: Identifies equipment type and brands with confidence scores
2. **Brand Recognition Focus**: Focuses on brand identification and facing counts

## 📁 Excel File Structure

**File**: `images (3).xlsx` (1,653 rows, 26 columns A-Z)

### Column Mapping
| Excel Column | Database Field | Purpose |
|--------------|----------------|---------|
| A | project_name | Project identifier |
| B | probe_id | **Unique key for duplicate detection** |
| C | image_taken_time | Date filtering |
| D | scene_id | Scene reference |
| E | visit_id | Visit reference |
| F | validated_by | Validation info |
| G | probe_status | Status info |
| H | template_id | Template reference |
| I | template_name | Template name |
| J | store_name | Store identifier |
| K | sales_rep_name | Sales rep info |
| L | sales_rep_id | Sales rep ID |
| M | comments_num | Comment count |
| N | creation_time | Creation timestamp |
| O | process_time | Processing time |
| P | validation_time | Validation timestamp |
| Q | validation_duration | Duration info |
| R | match_count | Match information |
| S | flag_time | Flag timestamp |
| T | original_width | Image dimensions |
| U | original_height | Image dimensions |
| V | client_version | Client info |
| W | user_agent | Browser info |
| X | probe_image_path | **S3 image URLs** |
| Y | flaw_types | Flaw information |
| Z | has_flaws | Flaw status |

## 🏗️ Application Architecture

### Page Structure
```
/
├── upload/          # Excel file upload page
├── gallery/         # Image gallery with filtering
├── instructions/    # Instruction management
├── analysis/        # Batch processing queue
└── results/         # Analysis results & export
```

### Component Hierarchy
```
App
├── Layout
│   ├── Navigation (multi-page)
│   └── Main Content
├── Upload Page
│   ├── FileUpload Component
│   ├── DuplicateDetection
│   └── ImportProgress
├── Gallery Page
│   ├── FilterBar (date range, processing status, instruction)
│   ├── ImageGrid (10 per page)
│   ├── Pagination
│   └── ImageModal
├── Instructions Page
│   ├── InstructionList
│   ├── InstructionForm (CRUD)
│   └── TemplateManager
├── Analysis Page
│   ├── ImageSelector
│   ├── BatchQueue
│   ├── ProgressTracker
│   └── QueueManager
└── Results Page
    ├── ResultsGrid
    ├── ExportOptions
    └── ResultsFilter
```

## 🎨 UI/UX Requirements

### Design Specifications
- **Layout**: Laptop-optimized (1200px+ width)
- **Grid**: 10 images per page in responsive grid
- **Navigation**: Top navigation bar with page links
- **Filtering**: Clean filter controls with clear labels
- **Progress**: Real-time progress bars for operations
- **Modals**: Image detail modals with metadata overlay

### Image Display
- **Thumbnails**: 300x300px with metadata overlay
- **Metadata Overlay**: Probe ID, store name, date
- **Processing Status**: Visual indicators (processed/not processed)
- **Selection**: Checkbox selection for batch processing

## 🔧 Technical Implementation

### 1. Project Setup
```bash
# Initialize Next.js project
npx create-next-app@latest endcap-app --typescript --tailwind --eslint --app

# Required dependencies
npm install @supabase/supabase-js @google/generative-ai xlsx lucide-react

# Development dependencies
npm install -D @types/node
```

### 2. Environment Variables
```env
NEXT_PUBLIC_SUPABASE_URL=https://ybzoioqgbvcxqiejopja.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=[get from Supabase]
GOOGLE_GEMINI_API_KEY=[get from Google AI Studio]
```

### 3. Supabase Configuration
```typescript
// lib/supabase.ts
import { createClient } from '@supabase/supabase-js'

const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL!
const supabaseKey = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!

export const supabase = createClient(supabaseUrl, supabaseKey)
```

### 4. Google Gemini Integration
```typescript
// lib/gemini.ts
import { GoogleGenerativeAI } from '@google/generative-ai'

const genAI = new GoogleGenerativeAI(process.env.GOOGLE_GEMINI_API_KEY!)

export async function analyzeImage(imageUrl: string, instruction: string) {
  const model = genAI.getGenerativeModel({ model: "gemini-2.5-flash" })
  // Implementation details...
}
```

## 📊 Data Flow

### Upload Process
1. User selects Excel file
2. Parse Excel using xlsx library
3. Validate column structure
4. Check for duplicates by `probe_id`
5. Skip duplicates, insert new records
6. Show import progress and summary

### Gallery Filtering
1. Base query: `SELECT * FROM images`
2. Apply filters:
   - Date range: `WHERE image_taken_time BETWEEN ? AND ?`
   - Processing status: `LEFT JOIN image_analysis_results`
   - Instruction filter: `WHERE instruction_id = ?`
3. Pagination: `LIMIT 10 OFFSET ?`

### Batch Processing
1. User selects images and instruction
2. Create processing queue
3. For each image:
   - Fetch image from S3 URL
   - Send to Gemini API with instruction
   - Parse structured JSON response
   - Store in `image_analysis_results`
4. Update progress in real-time
5. Handle errors and retries

## 🎛️ Filter Implementation

### Gallery Page Filters
```typescript
interface FilterState {
  dateRange: {
    start: Date | null
    end: Date | null
  }
  processingStatus: 'all' | 'processed' | 'not_processed'
  instructionId: number | null
}
```

### SQL Queries
```sql
-- Get filtered images with processing status
SELECT 
  i.*,
  CASE 
    WHEN iar.id IS NOT NULL THEN 'processed'
    ELSE 'not_processed'
  END as processing_status,
  inst.name as instruction_name
FROM images i
LEFT JOIN image_analysis_results iar ON i.id = iar.image_id
LEFT JOIN instructions inst ON iar.instruction_id = inst.id
WHERE 
  i.image_taken_time >= $1 
  AND i.image_taken_time <= $2
  AND ($3 = 'all' OR 
       ($3 = 'processed' AND iar.id IS NOT NULL) OR
       ($3 = 'not_processed' AND iar.id IS NULL))
  AND ($4 IS NULL OR iar.instruction_id = $4)
ORDER BY i.image_taken_time DESC
LIMIT 10 OFFSET $5;
```

## 🚀 Development Phases

### Phase 1: Project Setup & Navigation
- [ ] Initialize Next.js project with TypeScript
- [ ] Setup Tailwind CSS
- [ ] Configure Supabase client
- [ ] Create basic navigation structure
- [ ] Setup environment variables

### Phase 2: Upload System
- [ ] Create Excel upload component
- [ ] Implement xlsx parsing
- [ ] Add duplicate detection by probe_id
- [ ] Build import progress UI
- [ ] Handle error cases

### Phase 3: Image Gallery
- [ ] Create paginated image grid
- [ ] Implement date range filtering
- [ ] Add processing status filter
- [ ] Add instruction-based filtering
- [ ] Build image detail modal

### Phase 4: Instruction Management
- [ ] Create instruction CRUD interface
- [ ] Build instruction form
- [ ] Add template management
- [ ] Implement validation

### Phase 5: AI Processing
- [ ] Integrate Google Gemini API
- [ ] Build batch processing queue
- [ ] Implement progress tracking
- [ ] Add error handling and retries
- [ ] Create multiple queue support

### Phase 6: Results & Export
- [ ] Build results viewing interface
- [ ] Implement export functionality
- [ ] Add result filtering
- [ ] Create summary reports

### Phase 7: Testing & Polish
- [ ] Add comprehensive error handling
- [ ] Implement loading states
- [ ] Add success/error notifications
- [ ] Performance optimization
- [ ] Mobile responsiveness (if needed)

## 🔍 Quality Assurance

### Testing Checklist
- [ ] Excel upload with various file formats
- [ ] Duplicate detection accuracy
- [ ] Filter combinations work correctly
- [ ] Batch processing handles errors gracefully
- [ ] Large dataset performance (1,654+ images)
- [ ] API rate limiting handling
- [ ] Network error recovery

### Performance Considerations
- Image lazy loading in gallery
- Pagination for large datasets
- API request batching
- Error retry mechanisms
- Progress tracking optimization

## 📝 Documentation Requirements

### Code Documentation
- Component prop interfaces
- Function parameter types
- API endpoint documentation
- Database query explanations

### User Documentation
- Upload file format requirements
- Filter usage instructions
- Batch processing guidelines
- Export format specifications

## 🔐 Security Considerations

- Input validation for Excel files
- SQL injection prevention
- API key protection
- File size limits
- Rate limiting for API calls
- Error message sanitization

## 🚢 Deployment Notes

### Environment Setup
- Configure production Supabase keys
- Set up Google Gemini API keys
- Configure CORS policies
- Set up error logging

### Performance Monitoring
- Track API response times
- Monitor batch processing success rates
- Log error patterns
- Track user engagement metrics

---

## 📞 Support Information

**Supabase Project**: ybzoioqgbvcxqiejopja  
**Region**: eu-central-1  
**Database**: PostgreSQL 15.8.1.111  

This document serves as the complete specification for building the AI Image Processing App. Follow the phases sequentially and refer to this document for all implementation details.

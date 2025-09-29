# Endcap - AI Retail Image Analysis Platform

A production-ready AI-powered retail image analysis platform built with Next.js, Supabase, and Google Gemini 2.5 Flash API. Process thousands of retail images with custom AI instructions, advanced filtering, and comprehensive analytics.

**🌐 Live Application**: https://checkmyendcaps.vercel.app

## 🚀 Key Features

- **Store-Grouped Dashboard**: Organize 3,352+ images across 1,049 stores with intelligent grouping
- **Parallel AI Processing**: 8x faster analysis with concurrent Gemini API processing (up to 10 simultaneous)
- **Advanced Filtering**: Date range, search, analysis status, instruction type, and equipment type filtering
- **Real-time Analytics**: Live statistics showing processing progress and store coverage
- **Professional Authentication**: Supabase Auth with protected routes and user management
- **Excel Integration**: Smart upload with duplicate detection and batch processing
- **Equipment Classification**: Specialized endcap vs regular door detection with confidence scoring

## 🏗️ Tech Stack

- **Frontend**: Next.js 15.5.4, TypeScript, Tailwind CSS
- **Database**: Supabase PostgreSQL with real-time capabilities
- **AI Engine**: Google Gemini 2.5 Flash API with rate limit management
- **Authentication**: Supabase Auth (email/password)
- **File Processing**: xlsx library with batch validation
- **Deployment**: Vercel (production-ready)

## 📊 Production Statistics

- **Total Stores**: 1,049 unique retail locations
- **Total Images**: 3,352 retail images with S3 storage  
- **Processed Images**: 92 analyzed with AI (2.7% completion rate)
- **Analysis Success Rate**: 98%+ with parallel processing
- **Processing Speed**: 8x faster with parallel mode vs sequential

## 📖 Documentation

For complete development guide, system architecture, implementation details, and deployment instructions, see:

**[📋 COMPREHENSIVE-DEVELOPMENT-GUIDE.md](./COMPREHENSIVE-DEVELOPMENT-GUIDE.md)**

This guide covers:
- Complete system architecture and database schema
- Feature specifications and UI/UX design principles  
- Critical implementation learnings and troubleshooting
- Performance optimizations and rate limiting strategies
- Production deployment and monitoring guidelines

## 🚀 Quick Start

### Prerequisites
- Node.js 18+
- Supabase account  
- Google Gemini API key

### Installation
```bash
git clone https://github.com/paulalexeevich/endcap-ai-retail-analysis.git
cd endcap-ai-retail-analysis/ai-retail-app
npm install
```

### Environment Setup
```env
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key  
GOOGLE_GEMINI_API_KEY=your_gemini_api_key
```

### Development
```bash
npm run dev
# Open http://localhost:3000
```

## 🏗️ Application Architecture

### Core Pages
- **Dashboard** (`/results`): Store-grouped image management with advanced filtering
- **Analysis** (`/analysis`): Batch AI processing with parallel execution  
- **Instructions** (`/instructions`): AI prompt management and customization

### Key Components
```
ai-retail-app/
├── app/
│   ├── auth/              # Authentication pages (login/signup)
│   ├── results/           # Dashboard with store grouping and filtering
│   ├── analysis/          # Batch AI processing interface
│   └── instructions/      # AI prompt management
├── components/
│   ├── AuthProvider.tsx   # Supabase Auth integration
│   ├── Navigation.tsx     # Sidebar navigation with mobile support
│   └── ProtectedRoute.tsx # Route protection wrapper
└── lib/
    ├── supabase.ts        # Database client and type definitions
    └── gemini.ts          # AI processing with rate limiting
```

## 🤖 AI Processing Capabilities

### Gemini 2.5 Flash Integration
- **Parallel Processing**: Up to 10 concurrent API requests
- **Rate Limit Compliance**: Free tier (10 RPM, 250K TPM, 250 RPD)
- **Equipment Classification**: Endcap vs regular door detection
- **Brand Recognition**: Product brand identification with confidence scores
- **Flexible JSON Output**: Support for custom analysis structures

### Analysis Results Format
```json
{
  "equipment_type": { "type": "endcap", "confidence": 0.92 },
  "products": [{ "name": "Product", "brand": "Brand", "confidence": 0.95 }],
  "brands": [{ "name": "Brand Name", "confidence": 0.85 }],
  "summary": "Analysis summary",
  "timestamp": "2024-01-01T00:00:00.000Z"
}
```

## 🚀 Deployment

The application is deployed on Vercel with automatic deployments from the main branch:

```bash
# Deploy to production
npx vercel --prod
```

**Environment Variables Required:**
- `NEXT_PUBLIC_SUPABASE_URL`
- `NEXT_PUBLIC_SUPABASE_ANON_KEY`
- `GOOGLE_GEMINI_API_KEY`

## 🤝 Contributing

1. Fork the repository
2. Create feature branch: `git checkout -b feature/amazing-feature`
3. Commit changes: `git commit -m 'Add amazing feature'`
4. Push to branch: `git push origin feature/amazing-feature`
5. Open a Pull Request

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

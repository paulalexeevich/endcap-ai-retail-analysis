# Endcap - AI Retail Analysis Platform

A comprehensive AI-powered image processing and retail analysis application built with Next.js, Supabase, and Google Gemini API.

## Features

- **AI Image Processing**: Advanced product detection and classification using Google Gemini 2.5 Flash
- **Batch Analysis**: Process multiple images with structured JSON export
- **Interactive Visualization**: Split-screen layout with real-time analytics
- **Data Management**: Excel file upload with validation and duplicate detection
- **Retail Analytics**: Store distribution analysis and conversational AI interface

## Tech Stack

- **Frontend**: Next.js 14, TypeScript, Tailwind CSS
- **Backend**: Supabase (Database & Auth)
- **AI**: Google Gemini API, OpenAI GPT-3.5-turbo
- **Visualization**: Recharts
- **File Processing**: xlsx library

## Getting Started

### Prerequisites

- Node.js 18+ 
- npm or yarn
- Supabase account
- Google Gemini API key
- OpenAI API key (optional)

### Installation

1. Clone the repository:
```bash
git clone <your-repo-url>
cd Endcap
```

2. Install dependencies:
```bash
npm install
# or
yarn install
```

3. Set up environment variables:
```bash
cp .env.example .env.local
```

Fill in your API keys:
```env
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key
GOOGLE_GEMINI_API_KEY=your_gemini_api_key
OPENAI_API_KEY=your_openai_api_key
```

4. Run the development server:
```bash
npm run dev
# or
yarn dev
```

Open [http://localhost:3000](http://localhost:3000) with your browser to see the result.

## Project Structure

```
├── app/                    # Next.js 14 app directory
├── components/            # Reusable UI components
├── lib/                   # Utility functions and configurations
├── public/               # Static assets
├── supabase/             # Database migrations and types
└── types/                # TypeScript type definitions
```

## Key Features

### 1. Image Analysis Workflow
- Product detection with coordinate mapping
- Brand, flavor, and size classification
- Retailer search integration
- Interactive overlay visualization

### 2. Batch Processing
- Excel file upload and parsing
- Grid-based image selection
- Progress tracking for batch operations
- Structured JSON export with timestamps

### 3. Analytics Dashboard
- Real-time data visualization
- Interactive filtering by product, state, and time
- Context-aware AI responses
- Automatic chart generation

## API Endpoints

- `/api/analyze` - Single image analysis
- `/api/batch-analyze` - Batch image processing
- `/api/chat` - AI conversation interface

## Contributing

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## Environment Setup Notes

⚠️ **Important**: The server will return 500 errors without proper environment variable configuration. Ensure all required API keys are set before running the application.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

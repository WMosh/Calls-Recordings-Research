# Call Recording Management System

## Project Overview
Call Recording Management System is an advanced, privacy-focused solution for indexing, searching, and playing personal call recordings. Built for local-first deployment, this project is ideal for individuals, researchers, and professionals who need secure, high-performance access to large collections of call audio files on their own machines.

## Features- Automated scanning and indexing of thousands of AMR call recordings  
- Deep metadata extraction: date, time, duration, phone number, contact association, day of week, file size  
- Supports real VCF contact import and cross-referencing  
- Powerful multi-parameter search and filtering interface  
- Instant audio streaming in-browser, no cloud dependencies  
- Modern React + TypeScript frontend with Tailwind & shadcn/ui  
- Flask-based API and PostgreSQL storage, fully containerized with Docker  
- Architecture and code quality suitable for personal portfolios and technical interviews

## Demo

*(Insert a screenshot of the web interface here, e.g. Dashboard, Filters, Audio Player)*

## Quick Start

```bash
# 1. Clone the repository
git clone https://github.com/yourusername/call-recording-manager.git
cd call-recording-manager

# 2. Set up recordings directory in .env
echo "HOST_RECORDINGS_PATH=/absolute/path/to/your/recordings" > .env

# 3. Build and deploy with Docker
docker-compose up -d

# 4. Run database migrations
docker-compose exec backend flask db upgrade

# 5. Access frontend at http://localhost:3000
#    Access API docs at http://localhost:5000/api/docs
```

## Architecture
- Fully modular: React + Vite + shadcn/ui frontend | Flask + SQLAlchemy backend | PostgreSQL DB
- All components run locally, with Docker ensuring reproducible environments
- Audio files are accessed directly using absolute paths, guaranteeing privacy and efficiency

*(See full architecture diagram in /docs or project specification)*

## Technology Stack

| Layer       | Technology           |
|-------------|---------------------|
| Frontend    | React 18, TypeScript, Vite, Tailwind CSS, shadcn/ui |
| Backend     | Python 3.12, Flask, SQLAlchemy, FFmpeg, Pydub, vobject |
| Database    | PostgreSQL 16       |
| DevOps      | Docker, Docker Compose |

## Configuration
All configuration is managed with `.env` files.  
See the [specification](call-recordings-spec.md) or the included sample `.env` for details.  
Change default credentials before deployment.

## Contributing
Contributions are welcome!  
Please review `CONTRIBUTING.md` for guidelines and code of conduct.  
To propose new features, submit an issue or pull request.  
Tests must pass before merging.

## License
MIT License. See `LICENSE` for full terms.

## Credits
Developed by Moshe, incorporating best practices for privacy, modularity, and performance.  
Built as a showcase for technical interviews and portfolio use.

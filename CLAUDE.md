# GitHub Readme Stats - Project Guide for Claude

## Project Overview

GitHub Readme Stats is a service that dynamically generates customizable stats cards for GitHub profiles. It provides API endpoints that return SVG images displaying user statistics, top languages, repository cards, and more.

**Original Project:** [anuraghazra/github-readme-stats](https://github.com/anuraghazra/github-readme-stats)

## Architecture

### Technology Stack
- **Runtime:** Node.js 22+
- **Framework:** Express.js
- **Language:** JavaScript (ES Modules)
- **Deployment:** Docker + Docker Compose
- **Original Platform:** Vercel (serverless functions)

### Directory Structure
```
github-readme-stats/
├── api/                    # API endpoint handlers (Vercel functions)
│   ├── index.js           # Stats card endpoint
│   ├── pin.js             # Repository card endpoint
│   ├── top-langs.js       # Top languages card endpoint
│   ├── wakatime.js        # Wakatime stats endpoint
│   └── gist.js            # Gist card endpoint
├── src/
│   ├── cards/             # SVG card generators
│   ├── common/            # Shared utilities
│   ├── fetchers/          # GitHub API data fetchers
│   ├── themes/            # Theme configurations
│   └── index.js           # Module exports
├── themes/                 # Legacy themes directory (keep for compatibility)
├── tests/                  # Jest test suites
├── scripts/               # Utility scripts
├── express.js             # Express server wrapper (Docker entry point)
├── Dockerfile             # Docker configuration
└── docker-compose.yml     # Container orchestration
```

## Key Components

### 1. API Endpoints (api/)
Each file exports a default function that handles requests:
- Originally designed as Vercel serverless functions
- Now wrapped by Express server for self-hosting

### 2. Card Generators (src/cards/)
- Generate SVG cards with user statistics
- Support multiple themes and customization options
- Use D3-like data binding for SVG generation

### 3. Data Fetchers (src/fetchers/)
- Interact with GitHub GraphQL API
- Handle rate limiting and retries
- Cache responses for performance

### 4. Themes (src/themes/)
- Color schemes and styling configurations
- Extensible theme system
- Custom theme support via URL parameters

## Environment Variables

Create a `.env` file in the root directory:

```env
# GitHub Personal Access Tokens (for higher rate limits)
PAT_1=ghp_your_token_here
PAT_2=ghp_optional_second_token

# Optional: Multiple tokens for load balancing
# PAT_3=ghp_token
# PAT_4=ghp_token

# Server Configuration
PORT=3000
NODE_ENV=production
```

**Important:** Never commit `.env` file to version control.

## Docker Setup

### Development
```bash
# Build and start
docker compose up --build -d

# View logs
docker logs -f github-readme-stats

# Stop
docker compose down
```

### Production Deployment
```bash
# Build optimized image
docker compose -f docker-compose.yml up --build -d

# Check status
docker ps

# Monitor logs
docker logs --tail 100 -f github-readme-stats
```

## API Usage

### Stats Card
```
GET /api/?username=USERNAME
```

Optional parameters:
- `theme`: Theme name (default, dark, radical, etc.)
- `hide`: Comma-separated stats to hide (stars,commits,prs,issues)
- `show_icons`: Boolean to show/hide icons
- `title_color`, `icon_color`, `text_color`, `bg_color`: Custom colors

### Top Languages Card
```
GET /api/top-langs/?username=USERNAME
```

Optional parameters:
- `layout`: Layout style (default, compact)
- `langs_count`: Number of languages to show
- `hide`: Languages to exclude

### Repository Card
```
GET /api/pin/?username=USERNAME&repo=REPO_NAME
```

## Common Development Tasks

### Adding a New Theme
1. Edit `src/themes/index.js`
2. Add theme configuration to the themes object
3. Test with `?theme=your-theme-name`
4. Run tests: `npm test`

### Modifying Card Layouts
- Edit files in `src/cards/`
- SVG generation uses template literals
- Test locally before deploying

### Adding New Statistics
1. Update GraphQL query in `src/fetchers/stats.js`
2. Process data in card generator
3. Update tests and documentation

## Testing

```bash
# Run all tests
npm test

# Watch mode
npm run test:watch

# Update snapshots
npm run test:update:snapshot

# E2E tests
npm run test:e2e
```

## Known Issues & Solutions

### Docker Container Restart Loop
**Solved:** See [DOCKER_SETUP.md](DOCKER_SETUP.md) for complete documentation of the fix.

Summary of fixes:
- Fixed import paths in `src/common/color.js`
- Added missing dependencies to `src/package.json`
- Changed Dockerfile entry point to `express.js`
- Updated to Node.js 22
- Fixed build context in docker-compose.yml

### GitHub API Rate Limits
- Use Personal Access Tokens (PAT) to increase limits
- Configure multiple tokens for rotation
- Implement caching strategy

### Module Resolution
- Project uses ES modules (`"type": "module"` in package.json)
- All imports must include `.js` extension
- CommonJS `require()` won't work

## Code Style

### Imports
```javascript
// Always use .js extension
import { themes } from "../themes/index.js";

// Named imports preferred
import { renderError, renderStatsCard } from "./common/index.js";
```

### Async/Await
```javascript
// Use async/await instead of promises
const stats = await fetchStats(username);
```

### Error Handling
```javascript
// Use try/catch with proper error cards
try {
  const data = await fetchData();
  return renderCard(data);
} catch (error) {
  return renderError(error.message);
}
```

## Deployment Checklist

Before deploying to VPS:

- [ ] Set up environment variables (`.env` file)
- [ ] Configure GitHub Personal Access Tokens
- [ ] Update domain in `docker-compose.yml` (Traefik labels)
- [ ] Test locally with `docker compose up`
- [ ] Verify all API endpoints work
- [ ] Set up SSL/TLS certificates
- [ ] Configure reverse proxy (if not using Traefik)
- [ ] Set up monitoring and logging
- [ ] Configure automatic restarts (`restart: unless-stopped`)
- [ ] Set up backup strategy
- [ ] Document custom configurations

## Useful Commands

```bash
# Local development (without Docker)
npm install
node express.js

# Check Node version
node --version  # Should be 22+

# Format code
npm run format

# Lint code
npm run lint

# Docker cleanup
docker compose down -v  # Remove volumes
docker system prune -a  # Clean all unused resources

# View real-time logs
docker logs -f github-readme-stats

# Execute commands in container
docker exec -it github-readme-stats sh

# Rebuild specific service
docker compose up --build github-readme-stats
```

## Security Considerations

1. **API Tokens:** Never expose GitHub tokens in client-side code or logs
2. **Rate Limiting:** Implement rate limiting to prevent abuse
3. **Input Validation:** Sanitize all user inputs (usernames, repo names)
4. **CORS:** Configure appropriate CORS headers
5. **Updates:** Regularly update dependencies for security patches

## Performance Optimization

1. **Caching:** Implement Redis or memory cache for API responses
2. **CDN:** Use CDN for static assets and SVG responses
3. **Compression:** Enable gzip/brotli compression
4. **Connection Pooling:** Reuse HTTP connections to GitHub API
5. **Lazy Loading:** Only fetch data when needed

## Monitoring

Key metrics to monitor:
- Request rate and response times
- GitHub API rate limit usage
- Error rates by endpoint
- Container health and restarts
- Memory and CPU usage

## Support & Resources

- **Original Repository:** https://github.com/anuraghazra/github-readme-stats
- **Docker Documentation:** [DOCKER_SETUP.md](DOCKER_SETUP.md)
- **GitHub API Docs:** https://docs.github.com/en/graphql
- **Express.js Docs:** https://expressjs.com/

## Project Status

✅ **Working:** Docker containerization with local deployment
🔄 **In Progress:** VPS deployment preparation
📋 **Planned:** API vs Package decision for final deployment method

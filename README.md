# microSaaS Music Generation Platform

This project is a **microSaaS** that allows users to generate AI-created music from textual prompts.  The application is split into a **frontend** built with **React** and **Vite** and a **backend** built with **Node.js** and **Express**.  A **Supabase** PostgreSQL database stores user data and job statuses, and integration points with the **Suno API** generate music.  This repository uses a monorepo structure to keep the client and server in sync, and it is designed to be easily extendable with features like subscription billing and role-based access control.

## Features

- **AI-generated music**: Send prompts to the Suno API to generate two songs per request, with streaming and download URLs delivered through callbacks.
- **React/Vite frontend**: Fast development server and modern ES module support. Only environment variables prefixed with `VITE_` are exposed to the client (others remain private) [vite.dev](https://vite.dev/guide/env-and-mode#:~:text=Vite%20exposes%20env%20variables%20under,object%20as%20strings%20automatically).
- **Node/Express backend**: RESTful API with proper middlewares, validation, and error handling. Security best practices include using the **Helmet** middleware to set security-related headers [expressjs.com](https://expressjs.com/en/advanced/best-practice-security.html#:~:text=Use%20Helmet), rate limiting to prevent abuse [betterstack.com](https://betterstack.com/community/guides/scaling-nodejs/rate-limiting-express/#:~:text=Rate%20limiting%20is%20essential%20for,causing%20downtime%20for%20legitimate%20users), and optional slow-down logic to delay repeated requests [developer.mozilla.org](https://developer.mozilla.org/en-US/blog/securing-apis-express-rate-limit-and-slow-down/#:~:text=The%20express,when%20the%20limit%20is%20exceeded).
- **Supabase database**: Uses PostgreSQL and Prisma ORM for migrations and type-safe queries.
- **Polling job queue**: Jobs are stored in the database with a status of `pending`, and the frontend polls a `/status/:id` endpoint to check for completion. The Suno API’s concurrency limit (20 requests every 10 seconds) [docs.sunoapi.org](https://docs.sunoapi.org/suno-api/generate-music#:~:text=This%20is%20the%20key%20endpoint,request%20returns%20exactly%202%20songs) informs how many jobs can be processed concurrently.
- **CI/CD with GitHub Actions**: Separate workflows for backend and frontend. Workflows can be triggered only when files in a given path change using the `paths` or `paths-ignore` filters [docs.github.com](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#:~:text=A%20workflow%20with%20the%20following,the%20root%20of%20the%20repository).
- **AI collaboration guidelines**: The `ai/AI_INSTRUCTIONS.md` file describes how to interact with AI tools like Windsurf and Lovable.

## Repository structure

```
/
├── .github/workflows/        # GitHub Actions definitions
│   ├── ci-backend.yml        # CI/CD pipeline for backend (tests, lint, build)
│   └── ci-frontend.yml       # CI/CD pipeline for frontend (tests, lint, build)
├── backend/                  # Node.js + Express API
│   ├── src/
│   │   ├── controllers/      # Request handlers (thin, delegate to services)
│   │   ├── routes/           # Route definitions
│   │   ├── services/         # Business logic and data access
│   │   ├── models/           # Prisma models and database entities
│   │   ├── middlewares/      # Auth, validation, error handling, rate limiting
│   │   ├── jobs/             # Job queue logic for music generation
│   │   ├── config/           # App configuration & environment variables
│   │   ├── app.js            # Express app setup (middlewares, routes)
│   │   └── server.js         # Starts the HTTP server
│   ├── prisma/               # Prisma schema and migrations
│   ├── tests/                # Jest and SuperTest tests
│   ├── package.json          # Backend dependencies and scripts
│   ├── .env.example          # Example environment variables
│   └── README.md             # Backend-specific documentation
├── frontend/                 # React + Vite client
│   ├── public/               # Static assets
│   ├── src/
│   │   ├── components/       # Reusable UI components
│   │   ├── pages/            # Pages/views
│   │   ├── hooks/            # Custom React hooks
│   │   ├── context/          # React context for global state
│   │   ├── assets/           # Images, icons, fonts
│   │   ├── App.jsx           # Top-level component
│   │   └── main.jsx          # Vite entry point
│   ├── index.html            # HTML template
│   ├── vite.config.js        # Vite configuration
│   ├── package.json          # Frontend dependencies and scripts
│   ├── .env.example          # Example environment variables
│   └── README.md             # Frontend-specific documentation
├── ai/
│   └── AI_INSTRUCTIONS.md    # Guidelines for AI collaboration tools
└── README.md                 # This file

```

## Setup

### Prerequisites

- **Node.js** version 18 or later.
- **npm** (installed with Node). You can use **pnpm** or **yarn** if preferred.
- A **Supabase** project with a PostgreSQL database. Copy the URL and anon/key for your environment.
- (Optional) A **Redis** instance if you decide to use BullMQ for advanced job queues.
- (Optional) A secrets manager like **HashiCorp Vault** or **Infisical** for storing API keys. Avoid putting secrets directly into environment variables because they are poorly managed and not secure for production [nodejs-security.com](https://www.nodejs-security.com/blog/do-not-use-secrets-in-environment-variables-and-here-is-how-to-do-it-better#:~:text=If%20it%E2%80%99s%20a%20hobby%20project%2C,for%20you%20to%20do%20it) [nodejs-security.com](https://www.nodejs-security.com/blog/do-not-use-secrets-in-environment-variables-and-here-is-how-to-do-it-better#:~:text=To%20begin%20with%2C%20environment%20variables,Consider%20the%20following%20common%20pitfalls).

### Environment variables

Create a `.env` file in both `backend` and `frontend` directories based on the provided `.env.example` files.  **Do not commit your `.env` files to version control**.  Variables for the client must begin with `VITE_` to be exposed via `import.meta.env`; non-prefixed variables are kept on the server [vite.dev](https://vite.dev/guide/env-and-mode#:~:text=Vite%20exposes%20env%20variables%20under,object%20as%20strings%20automatically).  Example variables:

- For `backend/.env`:
    - `PORT=4000` – port for the API server.
    - `SUPABASE_URL=your_supabase_url` – your Supabase project URL.
    - `SUPABASE_KEY=your_supabase_key` – your Supabase service role key.
    - `SUNO_API_KEY=your_suno_api_key` – API key for Suno.
    - `JWT_SECRET=` – secret used for signing JWT tokens (if using authentication).
    - `CORS_ORIGIN=http://localhost:5173` – allowed origin for your frontend.
- For `frontend/.env` (prefix variables with `VITE_`):
    - `VITE_API_URL=http://localhost:4000` – base URL for API requests.
    - `VITE_SUPABASE_URL=your_supabase_url` – accessible in the client for Supabase auth.
    - `VITE_SOME_PUBLIC_KEY=` – any other safe value you need to expose.

If you need to store sensitive API keys or database passwords, use a secrets management service instead of environment variables to avoid leaks and ensure rotation and auditability [nodejs-security.com](https://www.nodejs-security.com/blog/do-not-use-secrets-in-environment-variables-and-here-is-how-to-do-it-better#:~:text=If%20it%E2%80%99s%20a%20hobby%20project%2C,for%20you%20to%20do%20it) [nodejs-security.com](https://www.nodejs-security.com/blog/do-not-use-secrets-in-environment-variables-and-here-is-how-to-do-it-better#:~:text=To%20begin%20with%2C%20environment%20variables,Consider%20the%20following%20common%20pitfalls).

### Installation

```bash
# Clone the repository and install dependencies
git clone <your-repo-url>
cd microSaaS-music

# Install backend dependencies
cd backend
npm install

# Install frontend dependencies
cd ../frontend
npm install

```

### Database

This project uses **Prisma** as an ORM.  To initialize the database schema:

```bash
cd backend
npx prisma migrate dev

```

Prisma will connect to your Supabase PostgreSQL database using the `DATABASE_URL` environment variable and run migrations automatically.  See `backend/prisma/schema.prisma` for model definitions and adjust as needed.

### Running the development servers

- **Backend**:
    
    ```bash
    cd backend
    npm run dev         # Start the Express server with nodemon or ts-node
    
    ```
    
    The server is configured in `src/app.js` and listens on the port defined in `.env`.  It automatically reloads on changes.
    
- **Frontend**:
    
    ```bash
    cd frontend
    npm run dev         # Start the Vite development server
    
    ```
    
    Vite serves the React application at http://localhost:5173 by default.  It proxies API requests to the backend if configured in `vite.config.js`.
    

### Production build

- **Backend**:
    
    ```bash
    cd backend
    npm run build       # Compile TypeScript or transpile code (if applicable)
    npm start           # Start the compiled server
    
    ```
    
- **Frontend**:
    
    ```bash
    cd frontend
    npm run build       # Build static files for production
    # Copy `dist/` to your preferred hosting platform (Netlify, Vercel, etc.)
    
    ```
    

### Testing

- **Backend tests** are written with **Jest** and **SuperTest**. Run them with:
    
    ```bash
    cd backend
    npm test
    
    ```
    
- **Frontend tests** use **React Testing Library** and **Jest**. Run them with:
    
    ```bash
    cd frontend
    npm test
    
    ```
    

Ensure that tests are part of your CI pipeline so that errors are caught early.

## Backend architecture

### App and server separation

The Express app is configured in `src/app.js`, where middlewares, routes and error handlers are registered.  The server in `src/server.js` simply imports the app and calls `app.listen()`.  This separation makes the app easier to test and more flexible for integration into serverless frameworks.

### Middlewares and security

- **Helmet**: Adds HTTP headers such as `Content-Security-Policy`, `Strict-Transport-Security`, `Referrer-Policy` and others to mitigate common vulnerabilities [expressjs.com](https://expressjs.com/en/advanced/best-practice-security.html#:~:text=Use%20Helmet). Install with `npm install helmet` and enable in `app.js`:
    
    ```jsx
    const helmet = require('helmet');
    app.use(helmet());
    
    ```
    
- **Rate limiting**: Prevent brute-force and DDoS attacks using `express-rate-limit`. Rate limiting protects your app from abuse and resource exhaustion [betterstack.com](https://betterstack.com/community/guides/scaling-nodejs/rate-limiting-express/#:~:text=Rate%20limiting%20is%20essential%20for,causing%20downtime%20for%20legitimate%20users). For example:
    
    ```jsx
    const rateLimit = require('express-rate-limit');
    const limiter = rateLimit({ windowMs: 15 * 60 * 1000, max: 100 });
    app.use(limiter);
    
    ```
    
- **Slow down**: Optionally introduce delays after repeated requests with `express-slow-down`. Unlike rate limiting, slow down middleware doesn’t reject requests; it adds a delay after a specified number of requests [developer.mozilla.org](https://developer.mozilla.org/en-US/blog/securing-apis-express-rate-limit-and-slow-down/#:~:text=The%20express,when%20the%20limit%20is%20exceeded). Use it to further discourage abuse:
    
    ```jsx
    const slowDown = require('express-slow-down');
    const speedLimiter = slowDown({ delayAfter: 10, delayMs: 500 });
    app.use(speedLimiter);
    
    ```
    
- **Body parsing and size limits**: Use `express.json({ limit: '1mb' })` to prevent large payloads from overwhelming the server.
- **Input validation**: Validate and sanitize request data using libraries like **express-validator** or **Zod**. Create reusable validation middlewares to parse and validate incoming bodies before they reach the controllers.
- **CORS**: Configure `cors` middleware to allow requests from your frontend domain.
- **Custom error handler**: Define a centralized error handler that formats errors consistently. You can create custom error classes (e.g., `AppError`) and throw them in your controllers.
- **Logging**: Use structured logging with libraries such as **Pino** or **Winston** to capture request/response details and application events.
- **Disable `x-powered-by` header**: Remove this header to reduce fingerprinting [expressjs.com](https://expressjs.com/en/advanced/best-practice-security.html#:~:text=It%20can%20help%20to%20provide,in%20the%20HTTP%20response%20headers):
    
    ```jsx
    app.disable('x-powered-by');
    
    ```
    

### Routes and services

Routes should be versioned (e.g., `/api/v1`) to make future changes backward-compatible.  Each route file registers endpoints and delegates to a controller.  Controllers are thin; they call services that encapsulate business logic and data access.

Example structure for a music generation endpoint:

```jsx
// routes/music.js
router.post('/generate', validateMusicRequest, musicController.generate);

// controllers/musicController.js
exports.generate = async (req, res, next) => {
  try {
    const { prompt, style, title } = req.body;
    const jobId = await musicService.enqueueJob({ prompt, style, title });
    res.status(202).json({ jobId });
  } catch (err) {
    next(err);
  }
};

// services/musicService.js
exports.enqueueJob = async ({ prompt, style, title }) => {
  // insert job into database with status 'pending'
  // optionally push to BullMQ queue here
  const job = await prisma.job.create({ data: { prompt, style, title, status: 'pending' } });
  // call Suno API
  return job.id;
};

```

### Job processing

The project uses a simple polling mechanism for the MVP.  When a user requests music generation, a job is created in the database with `status='pending'`.  A worker (could be a serverless function or a cron job) monitors jobs, sends requests to the Suno API and updates the job with the stream and download URLs once ready.  The frontend polls the `/status/:id` endpoint every few seconds to check progress.  The Suno API guarantees a streaming URL within 30–40 seconds and a downloadable URL within 2–3 minutes [docs.sunoapi.org](https://docs.sunoapi.org/suno-api/generate-music#:~:text=This%20is%20the%20key%20endpoint,request%20returns%20exactly%202%20songs).  Do not exceed the concurrency limit of **20 requests every 10 seconds** [docs.sunoapi.org](https://docs.sunoapi.org/suno-api/generate-music#:~:text=This%20is%20the%20key%20endpoint,request%20returns%20exactly%202%20songs); if you expect higher traffic, integrate a proper job queue such as BullMQ with Redis.

### Stripe integration (future work)

Stripe can be used for subscription billing.  To integrate:

1. Install the `stripe` npm package and store your API keys securely via a secrets manager.
2. Create endpoints to generate Checkout or Portal sessions (`/create-session`).
3. Handle webhooks on `/webhooks/stripe` and verify the signature of incoming events.
4. Store customer and subscription IDs in your database to manage access.

For the MVP, Stripe is not enabled.  When added, ensure idempotency keys and proper error handling for retries.

### Suno API integration

Send POST requests to the Suno API’s `/api/v1/generate` endpoint with the user’s prompt, style and other parameters.  The API returns exactly **two songs** per request and provides a streaming URL in about 30–40 seconds and a downloadable URL in 2–3 minutes [docs.sunoapi.org](https://docs.sunoapi.org/suno-api/generate-music#:~:text=This%20is%20the%20key%20endpoint,request%20returns%20exactly%202%20songs).  Respect the concurrency limit of **20 requests per 10 seconds** [docs.sunoapi.org](https://docs.sunoapi.org/suno-api/generate-music#:~:text=This%20is%20the%20key%20endpoint,request%20returns%20exactly%202%20songs).  After dispatching the request, store the resulting `taskId` in the job record and update the status when the callback arrives.

Use `axios` or `fetch` to call the Suno API.  Add retry logic and handle errors gracefully.  Validate and sanitize prompts before sending them to avoid API errors.

## Frontend architecture

### Structure

The React application uses Vite for a fast development experience and ES module support.  Organize your code into `components`, `pages`, `hooks` and `context` directories.  Each component can have its own folder with the component file, styles, tests and documentation.

The entry point (`main.jsx`) mounts the root `App` component.  The `App.jsx` component defines routes using React Router (if included) and global providers (e.g., context providers).  Use `.env` variables prefixed with `VITE_` to configure the API URL and other public settings [vite.dev](https://vite.dev/guide/env-and-mode#:~:text=Vite%20exposes%20env%20variables%20under,object%20as%20strings%20automatically).

### Key features (planned extensions)

- **Authentication**: Implement JWT or session-based authentication. Store tokens in HTTP-only cookies for security.
- **Subscriptions and payments**: Redirect users to Stripe Checkout or the customer portal for plan management.
- **Prompt form**: Allow users to input text and select style parameters. On submission, call the backend `/generate` endpoint and display job status.
- **Status polling**: Poll the API for job completion and show progress indicators. Once the streaming URL is available, embed an audio player. Provide a download link when the file is ready.
- **User dashboard**: Display subscription status, remaining credits and generation history.

### Testing and code quality

Use **React Testing Library** to test components in isolation.  Set up **eslint** and **prettier** for code quality, and run linters as part of your CI pipeline.  Optionally use **Vitest** for fast unit tests.

## CI/CD

This monorepo uses **GitHub Actions** for continuous integration and deployment.  Two workflow files (`ci-backend.yml` and `ci-frontend.yml`) live in `.github/workflows/`.  Each workflow runs tests, linters and builds only when relevant files change.  For example, the backend workflow might be triggered on pushes to the repository when files in `backend/**` change:

```yaml
on:
  push:
    paths:
      - 'backend/**'
      - '.github/workflows/ci-backend.yml'

```

Path filters allow you to selectively run workflows only when specified files are modified.  The official GitHub documentation explains how to include and exclude paths, noting that you cannot use `paths` and `paths-ignore` together for the same event.  Use the `!` prefix to exclude certain patterns and ensure at least one positive pattern is present [docs.github.com](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#:~:text=A%20workflow%20with%20the%20following,the%20root%20of%20the%20repository).

You can cache dependencies using the `actions/cache` action to speed up builds.  Use matrices to test against multiple Node.js versions or to run jobs in parallel.

## Best practices and further considerations

- **API versioning**: Include the API version in your route prefix (e.g., `/api/v1`). This practice makes it easier to introduce breaking changes in the future.
- **Health checks**: Add `/health` endpoints to check connectivity with the database, Redis and external services. Use these endpoints for readiness and liveness probes in production.
- **Monitoring and logging**: Use services like Better Stack, Grafana or Prometheus for observability. Structured logs (JSON) make it easier to parse and analyze logs.
- **Secure cookies**: Set `httpOnly`, `secure`, `sameSite` attributes on cookies when implementing authentication [expressjs.com](https://expressjs.com/en/advanced/best-practice-security.html#:~:text=Use%20Helmet).
- **Input sanitization**: Always validate and sanitize inputs to prevent injections. Use libraries like Zod or express-validator.
- **HTTPS**: Enforce TLS (HTTPS) in production to encrypt data in transit [expressjs.com](https://expressjs.com/en/advanced/best-practice-security.html#:~:text=Use%20TLS).
- **Idempotency and retries**: When dealing with payment systems or external APIs, implement idempotent request handling and retry strategies.
- **Internationalization**: Plan for multi-language support and multi-currency billing if you intend to serve global users.

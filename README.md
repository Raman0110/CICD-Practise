# CI/CD Practise

A small Express and TypeScript app for practising CI/CD.

## Requirements

- Node.js 24 and npm
- Git
- GitHub and Render accounts for the deployment sections

## Get started

Start the development server:

```bash
npm run dev
```

Open [localhost:3000](http://localhost:3000). The [health endpoint](http://localhost:3000/api/health) returns HTTP 200 with:

```json
{"status":"ok"}
```

Stop the server with Ctrl+C.

## Scripts

- `npm run dev`: start the development server with automatic restarts.
- `npm run typecheck`: check TypeScript without generating output.
- `npm test`: run the health endpoint test. The test starts and stops its own server.
- `npm run build`: compile the app into `dist`.
- `npm start`: run the compiled app after building it.

## Configuration

The server uses port `3000` unless `PORT` is set. The home page displays `APP_VERSION`, defaulting to `development`. Neither variable is required to run the starter locally.

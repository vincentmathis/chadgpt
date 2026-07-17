# ChadGPT

An AI chat interface with personality. Two personas: ChadGPT (a brutally honest looksmaxxing coach) and Winkler (Rainer Winkler in Bavarian dialect). Powered by Groq for fast streaming responses.

**Live:** [chadgpt.vincentmathis.xyz](https://chadgpt.vincentmathis.xyz)

## Tech Stack

- [SvelteKit](https://kit.svelte.dev) + [Svelte 5](https://svelte.dev)
- [Groq SDK](https://console.groq.com) for AI completions
- [Tailwind CSS v4](https://tailwindcss.com)
- [Biome](https://biomejs.dev) for linting/formatting

## Setup

Create a `.env` file with your Groq API key:

```
GROQ_API_KEY=your_key_here
```

Get a free key at [console.groq.com](https://console.groq.com).

## Development

```bash
bun install
bun run dev
```

## Building

```bash
bun run build
bun run preview
```

## Docker

```bash
docker build -t chadgpt .
docker run -p 3000:3000 -e GROQ_API_KEY=your_key chadgpt
```

## License

GPL-3.0

# Templated Generation (Svelte)

A simple web app that shows templated generation using LLMs (OpenRouter). 
Completely barebones svelte application, run `npm install` and `npm run dev` to get started.

Mess with the templates - copy the example to `templates.json`. Currently anything in the `{}` will get substituted with user input.
Make sure you also put in the API keys into a `.env`. e.g.:

```
// .env
VITE_OPENROUTER_API_KEY=...
```
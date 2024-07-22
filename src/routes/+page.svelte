<script lang="ts">
	import { writable } from "svelte/store";
	import { fade, fly } from "svelte/transition";
	import { cubicOut } from "svelte/easing";
	import { ProgressBar } from "@skeletonlabs/skeleton";
	import { marked } from 'marked';

	let userInput = "";
	let selectedTemplate = "";
	let output = writable("");
	let loading = writable(false);
	let templateSelected = false;
	let typingOutput = "";
	let typingIndex = 0;
	let typingTimer: NodeJS.Timeout;
	let filledTemplateOutput = "";
	let formattedOutput: string | Promise<string> = "";
	let previousGeneration = "";

	import templatesData from "../templates.json";

	const templates: { [key: string]: string } = Object.fromEntries(
		templatesData.templates.map(template => Object.entries(template)[0])
	);

	async function callOpenRouterAPI(prompt: string, previousMessage: string = ""): Promise<string> {
		const apiKey = import.meta.env.VITE_OPENROUTER_API_KEY;
		const siteUrl = import.meta.env.VITE_YOUR_SITE_URL;
		const siteName = import.meta.env.VITE_YOUR_SITE_NAME;

		if (!apiKey) {
			console.error(
				"API key not found. Please set VITE_OPENROUTER_API_KEY in your .env file.",
			);
			return "Error: API key not configured.";
		}

		try {
			const messages = previousMessage 
				? [
					{ role: "assistant", content: previousMessage },
					{ role: "user", content: "Please continue and expand upon this, incorporating the following ideas: " + prompt }
				]
				: [{ role: "user", content: prompt }];

			const response = await fetch(
				"https://openrouter.ai/api/v1/chat/completions",
				{
					method: "POST",
					headers: {
						Authorization: `Bearer ${apiKey}`,
						"HTTP-Referer": siteUrl,
						"X-Title": siteName,
						"Content-Type": "application/json",
					},
					body: JSON.stringify({
						model: "meta-llama/llama-3-8b-instruct",
						messages: messages,
					}),
				},
			);

			if (!response.ok) {
				throw new Error(`API request failed: ${response.statusText}`);
			}

			const data = await response.json();
			return data.choices[0].message.content;
		} catch (error: any) {
			console.error("Error calling OpenRouter API:", error);
			return `Error: ${error.message}`;
		}
	}

	let timer: NodeJS.Timeout;

	function debounce(func: Function, delay: number) {
		clearTimeout(timer);
		timer = setTimeout(() => func(), delay);
	}

	$: {
		if (userInput && templateSelected) {
			debounce(generateOutput, 2000);
		}
	}

	async function generateOutput() {
		$loading = true;
		$output = "Processing...";

		try {
			// Fill the template
			let filledTemplate = templates[selectedTemplate].replace(
				/\{(\w+)\}/g,
				(_, key) => {
					return userInput; // For simplicity, we're using the same input for all placeholders
				},
			);
			filledTemplateOutput = filledTemplate;

			// Call the API with the previous generation
			const enhancedOutput = await callOpenRouterAPI(filledTemplate, previousGeneration);

			$output = enhancedOutput;
			startTypingAnimation(enhancedOutput);

			// Store the current generation for the next iteration
			previousGeneration = enhancedOutput;
		} catch (error: any) {
			$output = `Error: ${error.message}`;
		} finally {
			$loading = false;
		}
	}

	function startTypingAnimation(text: string) {
		typingOutput = "";
		typingIndex = 0;
		clearInterval(typingTimer);
		typingTimer = setInterval(() => {
			if (typingIndex < text.length) {
				typingOutput += text[typingIndex];
				typingIndex++;
				formattedOutput = marked(typingOutput);
			} else {
				clearInterval(typingTimer);
			}
		}, 10);
	}

	function handleTemplateSelection() {
		templateSelected = true;
		previousGeneration = ""; // Reset previous generation when template changes
	}
</script>

<main class="mcontainer mx-auto max-w-screen-md p-4">
	{#if !templateSelected}
		<div class="h-64 flex justify-center items-center" in:fade>
			<div class="flex justify-center items-start flex-col grow">
				<p class="my-3">Templates × LLMs = 🔥</p>
				<select
					bind:value={selectedTemplate}
					on:change={handleTemplateSelection}
					class="p-2 border rounded text-lg select"
				>
					<option value="" disabled selected>Select a template</option>
					<option value="story">Short Story</option>
					<option value="product">New Product</option>
					<option value="fd">Functional Description</option>
					<option value="quote">Quote</option>
					<option value="article">Blog Post</option>
				</select>
			</div>
		</div>
	{:else}
		<div in:fly={{ y: -50, duration: 500, easing: cubicOut }} class="mb-4 mt-8">
			<select
				bind:value={selectedTemplate}
				class="p-2 border rounded select"
			>
			<option value="story">Short Story</option>
			<option value="product">New Product</option>
			<option value="fd">Functional Description</option>
			<option value="quote">Quote</option>
			<option value="article">Blog Post</option>
			</select>
		</div>

		<div class="flex gap-4" in:fade={{ duration: 300 }}>
			<div class="w-1/2">
				<textarea
					bind:value={userInput}
					disabled={$loading}
					class="w-full p-2 border rounded textarea"
					rows="10"
					placeholder="Enter some ideas here..."
				></textarea>
				{#if $loading}
					<ProgressBar value={undefined} />
				{/if}
			</div>

			<div class="w-1/2">
				<div class="border rounded p-4 h-full">
					{#if $loading}
						<div class="placeholder" in:fade={{ duration: 300 }}/>
					{:else}
						{@html formattedOutput}
					{/if}
				</div>
			</div>
		</div>
	{/if}
</main>

<style>
	@keyframes loading {
		0% {
			width: 0%;
		}
		100% {
			width: 100%;
		}
	}
</style>
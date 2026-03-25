<script lang="ts">
	import { browser } from '$app/environment';

	type DrawerProps = {
		open?: boolean;
		fileKey?: string;
		title?: string;
		onClose?: () => void;
	};

	const markdownModules = import.meta.glob('../assets/*.md', {
		query: '?raw',
		import: 'default'
	});
	const imageModules = import.meta.glob('../assets/**/*.{png,jpg,jpeg,gif,svg,webp,avif}', {
		eager: true,
		import: 'default'
	});

	let { open = false, fileKey = '', title = '', onClose = () => {} }: DrawerProps = $props();

	let loading = $state(false);
	let renderedHtml = $state('');
	let errorMessage = $state('');

	function resolveAssetPath(source: string, markdownKey: string): string {
		if (
			!source ||
			source.startsWith('http://') ||
			source.startsWith('https://') ||
			source.startsWith('data:') ||
			source.startsWith('/')
		) {
			return source;
		}

		const baseUrl = new URL(`https://example.invalid/assets/${markdownKey}.md`);
		const resolvedUrl = new URL(source, baseUrl);
		const relativePath = decodeURIComponent(resolvedUrl.pathname.replace(/^\/assets\//, ''));
		const assetKey = `../assets/${relativePath}`;

		return (imageModules[assetKey] as string | undefined) ?? source;
	}

	function rewriteImageSources(html: string, markdownKey: string): string {
		if (!browser) {
			return html;
		}

		const container = document.createElement('div');
		container.innerHTML = html;

		for (const image of container.querySelectorAll('img')) {
			const source = image.getAttribute('src') ?? '';
			image.setAttribute('src', resolveAssetPath(source, markdownKey));
			image.setAttribute('loading', 'lazy');
		}

		return container.innerHTML;
	}

	$effect(() => {
		let cancelled = false;

		async function loadMarkdown() {
			if (!browser || !open || !fileKey) {
				renderedHtml = '';
				errorMessage = '';
				loading = false;
				return;
			}

			const loader = markdownModules[`../assets/${fileKey}.md`];
			if (!loader) {
				renderedHtml = '';
				errorMessage = `Missing markdown file: ${fileKey}.md`;
				loading = false;
				return;
			}

			loading = true;
			errorMessage = '';

			try {
				const [markdown, markedModule, domPurifyModule] = await Promise.all([
					loader() as Promise<string>,
					import('marked'),
					import('dompurify')
				]);
				const rendered = await markedModule.marked.parse(markdown);
				const sanitized = domPurifyModule.default.sanitize(String(rendered));
				const withResolvedImages = rewriteImageSources(sanitized, fileKey);

				if (!cancelled) {
					renderedHtml = withResolvedImages;
				}
			} catch (error) {
				if (!cancelled) {
					renderedHtml = '';
					errorMessage = error instanceof Error ? error.message : 'Failed to load markdown.';
				}
			} finally {
				if (!cancelled) {
					loading = false;
				}
			}
		}

		loadMarkdown();

		return () => {
			cancelled = true;
		};
	});
</script>

<aside class:open class="drawer" aria-hidden={!open}>
	{#if open}
		<div class="drawer-shell">
			<div class="drawer-header">
				<div class="drawer-copy">
					<!-- <p class="drawer-label">Supplement</p> -->
					<h2>{title}</h2>
					<!-- <p class="drawer-file">{fileKey}.md</p> -->
				</div>

				<button class="drawer-close" type="button" onclick={onClose} aria-label="Close drawer">
					Close
				</button>
			</div>

			<div class="drawer-content">
				{#if loading}
					<p class="drawer-state">Loading markdown...</p>
				{:else if errorMessage}
					<p class="drawer-state">{errorMessage}</p>
				{:else}
					<div class="markdown-body">
						{@html renderedHtml}
					</div>
				{/if}
			</div>
		</div>
	{/if}
</aside>

<style>
	.drawer {
		display: none;
	}

	.drawer.open {
		display: block;
		position: sticky;
		top: var(--drawer-top, 72px);
		height: calc(100vh - var(--drawer-top, 72px) - 16px);
		align-self: start;
	}

	.drawer-shell {
		display: flex;
		flex-direction: column;
		height: 100%;
		background: #fff;
		border: 1px solid #e5e7eb;
		border-radius: 12px;
		overflow: hidden;
	}

	.drawer-header {
		display: flex;
		align-items: start;
		justify-content: space-between;
		gap: 12px;
		padding: 16px 16px 12px;
		border-bottom: 1px solid #e5e7eb;
		background: #fff;
	}

	.drawer-label,
	.drawer-file {
		margin: 0;
		font-size: 0.74rem;
		line-height: 1.2;
		color: #666;
	}

	.drawer-label {
		text-transform: uppercase;
		letter-spacing: 0.08em;
	}

	.drawer-header h2 {
		margin: 4px 0;
		font-size: 1.05rem;
		line-height: 1.2;
		font-weight: 600;
	}

	.drawer-close {
		border: 1px solid #d4d4d8;
		background: #fff;
		border-radius: 999px;
		padding: 6px 10px;
		font: inherit;
		cursor: pointer;
	}

	.drawer-content {
		flex: 1;
		overflow: auto;
		padding: 16px;
	}

	.drawer-state {
		margin: 0;
		font-size: 0.92rem;
		line-height: 1.5;
		color: #444;
	}

	.markdown-body {
		font-size: 0.95rem;
		line-height: 1.6;
		color: #222;
	}

	.markdown-body :global(h1),
	.markdown-body :global(h2),
	.markdown-body :global(h3) {
		margin: 0 0 0.75rem;
		line-height: 1.2;
	}

	.markdown-body :global(p),
	.markdown-body :global(ul),
	.markdown-body :global(ol) {
		margin: 0 0 0.9rem;
	}

	.markdown-body :global(code) {
		font-family: monospace;
		font-size: 0.9em;
	}

	.markdown-body :global(pre) {
		margin: 0 0 1rem;
		padding: 12px;
		overflow: auto;
		background: #f5f5f5;
		border-radius: 8px;
	}

	.markdown-body :global(img) {
		display: block;
		max-width: 100%;
		height: auto;
		margin: 0 0 1rem;
		border-radius: 8px;
	}

	@media (max-width: 960px) {
		.drawer.open {
			position: static;
			height: auto;
		}

		.drawer-shell {
			max-height: none;
		}
	}
</style>

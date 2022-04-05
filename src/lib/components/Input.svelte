<script>
	import Explainer from './Explainer.svelte';

	export let label;
	export let name;
	export let explainer = '';
	export let link = '';
</script>

<div class="mt-2 grid items-center sm:grid-cols-2">
	<div class="flex flex-col">
		<label for={name} class="mb-2 text-base font-bold text-stone-100 sm:mb-0">{label}</label>
		{#if explainer}
			<Explainer text={explainer} />
		{/if}
	</div>
	{#if link}
		<a class="no-underline" href={link}>
			<input id={name} {name} {...$$restProps} />
		</a>
	{:else}
		<input id={name} {name} {...$$restProps} required />{/if}
</div>

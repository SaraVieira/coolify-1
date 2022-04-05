<script lang="ts">
	export let gitSource;
	import { goto } from '$app/navigation';
	import { post } from '$lib/api';
	import Input from '$lib/components/Input.svelte';

	import { errorNotification } from '$lib/form';
	import { onMount } from 'svelte';

	let nameEl;

	onMount(() => {
		nameEl.focus();
	});
	async function handleSubmit() {
		try {
			const { id } = await post(`/new/source.json`, { ...gitSource });
			return await goto(`/sources/${id}/`);
		} catch ({ error }) {
			return errorNotification(error);
		}
	}
</script>

<div class="flex justify-center pb-8">
	<form on:submit|preventDefault={handleSubmit} class="grid grid-flow-row gap-2 py-4">
		<div class="flex h-8 items-center space-x-2">
			<div class="text-xl font-bold text-white">Configuration</div>
			<button type="submit" class="bg-orange-600 hover:bg-orange-500">Save</button>
		</div>
		<div class="grid items-center sm:grid-cols-2">
			<label for="type" class="text-base font-bold text-stone-100">Type</label>
			<select name="type" id="type" bind:value={gitSource.type}>
				<option value="github">GitHub</option>
				<option value="gitlab">GitLab</option>
				<option value="bitbucket">BitBucket</option>
			</select>
		</div>
		<Input
			label="Name"
			name="name"
			placeholder="GitHub.com"
			required
			bind:this={nameEl}
			bind:value={gitSource.name}
		/>
		<Input
			label="HTML URL"
			type="url"
			name="htmlUrl"
			placeholder="eg: https://github.com"
			required
			bind:value={gitSource.htmlUrl}
		/>

		<Input
			label="API URL"
			name="apiUrl"
			type="url"
			placeholder="eg: https://api.github.com"
			required
			bind:value={gitSource.apiUrl}
		/>
	</form>
</div>

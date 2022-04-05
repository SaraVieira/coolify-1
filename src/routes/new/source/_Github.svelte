<script lang="ts">
	export let gitSource;
	import { goto } from '$app/navigation';
	import { post } from '$lib/api';

	import Explainer from '$lib/components/Explainer.svelte';
	import Input from '$lib/components/Input.svelte';

	import { errorNotification } from '$lib/form';
	import { onMount } from 'svelte';

	let nameEl;
	let organizationEl;

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

<div class="mx-auto max-w-4xl px-6">
	<div class="flex justify-center pb-8">
		<form on:submit|preventDefault={handleSubmit} class="grid grid-flow-row gap-2 py-4 ">
			<div class="mb-6 flex h-8 items-center space-x-2 sm:mb-0">
				<div class="text-xl font-bold text-white ">Configuration</div>
				<button type="submit" class="bg-orange-600 hover:bg-orange-500">Save</button>
			</div>
			<div class="grid items-center sm:grid-cols-2 sm:px-10">
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
			<Input
				label="Organization"
				explainer="Fill it if you would like to use an organization's as your Git Source. Otherwise your user will be used."
				name="organization"
				placeholder="eg: coollabsio"
				bind:value={gitSource.organization}
				bind:this={organizationEl}
			/>
		</form>
	</div>
</div>

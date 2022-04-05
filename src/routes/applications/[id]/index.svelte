<script context="module" lang="ts">
	import type { Load } from '@sveltejs/kit';
	export const load: Load = async ({ fetch, params, stuff }) => {
		if (stuff?.application?.id) {
			return {
				props: {
					application: stuff.application,
					isRunning: stuff.isRunning
				}
			};
		}
		const endpoint = `/applications/${params.id}.json`;
		const res = await fetch(endpoint);

		if (res.ok) {
			return {
				props: {
					...(await res.json())
				}
			};
		}

		return {
			status: res.status,
			error: new Error(`Could not load ${endpoint}`)
		};
	};
</script>

<script lang="ts">
	export let application: Prisma.Application & {
		settings: Prisma.ApplicationSettings;
		gitlabApp: Prisma.GitlabApp;
		gitSource: Prisma.GitSource;
		destinationDocker: Prisma.DestinationDocker;
	};
	export let isRunning;
	import { page, session } from '$app/stores';
	import { errorNotification } from '$lib/form';
	import { onMount } from 'svelte';
	import Select from 'svelte-select';

	import Explainer from '$lib/components/Explainer.svelte';
	import Setting from '$lib/components/Setting.svelte';
	import type Prisma from '@prisma/client';
	import { notNodeDeployments, staticDeployments } from '$lib/components/common';
	import { toast } from '@zerodevx/svelte-toast';
	import { post } from '$lib/api';
	import cuid from 'cuid';
	import { browser } from '$app/env';
	import Input from '$lib/components/Input.svelte';
	import Github from '$lib/components/svg/Destinations/Github.svelte';
	import Gitlab from '$lib/components/svg/Destinations/Gitlab.svelte';
	import NewTab from '$lib/components/svg/common/NewTab.svelte';
	const { id } = $page.params;

	let domainEl: HTMLInputElement;

	let loading = false;
	let forceSave = false;
	let debug = application.settings.debug;
	let previews = application.settings.previews;
	let dualCerts = application.settings.dualCerts;
	let autodeploy = application.settings.autodeploy;

	let wsgis = [
		{
			value: 'None',
			label: 'None'
		},
		{
			value: 'Gunicorn',
			label: 'Gunicorn'
		}
		// },
		// {
		// 	value: 'uWSGI',
		// 	label: 'uWSGI'
		// }
	];

	if (browser && window.location.hostname === 'demo.coolify.io' && !application.fqdn) {
		application.fqdn = `http://${cuid()}.demo.coolify.io`;
	}

	onMount(() => {
		domainEl.focus();
	});

	async function changeSettings(name) {
		if (name === 'debug') {
			debug = !debug;
		}
		if (name === 'previews') {
			previews = !previews;
		}
		if (name === 'dualCerts') {
			dualCerts = !dualCerts;
		}
		if (name === 'autodeploy') {
			autodeploy = !autodeploy;
		}
		try {
			await post(`/applications/${id}/settings.json`, {
				previews,
				debug,
				dualCerts,
				autodeploy,
				branch: application.branch,
				projectId: application.projectId
			});
			return toast.push('Settings saved.');
		} catch ({ error }) {
			if (name === 'debug') {
				debug = !debug;
			}
			if (name === 'previews') {
				previews = !previews;
			}
			if (name === 'dualCerts') {
				dualCerts = !dualCerts;
			}
			if (name === 'autodeploy') {
				autodeploy = !autodeploy;
			}
			return errorNotification(error);
		}
	}
	async function handleSubmit() {
		loading = true;
		try {
			await post(`/applications/${id}/check.json`, { fqdn: application.fqdn, forceSave });
			await post(`/applications/${id}.json`, { ...application });
			return window.location.reload();
		} catch ({ error }) {
			if (error?.startsWith('DNS not set')) {
				forceSave = true;
			}
			return errorNotification(error);
		} finally {
			loading = false;
		}
	}
	async function selectWSGI(event) {
		application.pythonWSGI = event.detail.value;
	}
</script>

<div class="flex items-center space-x-2 p-5 px-6 font-bold">
	<div class="-mb-5 flex-col">
		<div class="md:max-w-64 truncate text-base tracking-tight md:text-2xl lg:block">
			Configuration
		</div>
		<span class="text-xs">{application.name} </span>
	</div>

	{#if application.fqdn}
		<a
			href={application.fqdn}
			target="_blank"
			class="icons tooltip-bottom flex items-center bg-transparent text-sm"><NewTab /></a
		>
	{/if}
	<a
		href="{application.gitSource.htmlUrl}/{application.repository}/tree/{application.branch}"
		target="_blank"
		class="w-10"
	>
		{#if application.gitSource?.type === 'gitlab'}
			<Gitlab />
		{:else if application.gitSource?.type === 'github'}
			<Github />
		{/if}
	</a>
</div>

<div class="mx-auto max-w-4xl px-1 sm:px-6">
	<!-- svelte-ignore missing-declaration -->
	<form on:submit|preventDefault={handleSubmit} class="py-4">
		<div class="flex space-x-1 pb-5 font-bold">
			<div class="title">General</div>
			{#if $session.isAdmin}
				<button
					type="submit"
					class:bg-green-600={!loading}
					class:bg-orange-600={forceSave}
					class:hover:bg-green-500={!loading}
					class:hover:bg-orange-400={forceSave}
					disabled={loading}
					>{loading ? 'Saving...' : forceSave ? 'Are you sure to continue?' : 'Save'}</button
				>
			{/if}
		</div>
		<div class="grid grid-flow-row gap-2 px-2 sm:px-10">
			<Input
				readonly={!$session.isAdmin}
				name="name"
				bind:value={application.name}
				required
				label="Name"
			/>
			<Input
				label="Git Source"
				value={application.gitSource.name}
				name="gitSource"
				disabled
				class="cursor-pointer hover:bg-coolgray-500"
				link={$session.isAdmin
					? `/applications/${id}/configuration/source?from=/applications/${id}`
					: ''}
			/>
			<Input
				label="Git Repository"
				value="{application.repository}/{application.branch}"
				name="repository"
				disabled
				class="cursor-pointer hover:bg-coolgray-500"
				link={$session.isAdmin
					? `/applications/${id}/configuration/repository?from=/applications/${id}&to=/applications/${id}/configuration/buildpack`
					: ''}
			/>

			<Input
				value={application.buildPack}
				name="buildPack"
				disabled
				class="cursor-pointer hover:bg-coolgray-500"
				label="Build Pack"
				link={$session.isAdmin
					? `/applications/${id}/configuration/buildpack?from=/applications/${id}`
					: ''}
			/>

			<Input
				value={application.destinationDocker.name}
				name="destination"
				disabled
				label="Destination"
				class="bg-transparent "
			/>
		</div>
		<div class="flex space-x-1 py-5 font-bold">
			<div class="title">Application</div>
		</div>
		<div class="grid grid-flow-row gap-2 px-2 sm:px-10">
			<Input
				label="URL (FQDN)"
				readonly={!$session.isAdmin || isRunning}
				disabled={!$session.isAdmin || isRunning}
				bind:this={domainEl}
				name="fqdn"
				bind:value={application.fqdn}
				pattern="^https?://([a-z0-9]+(-[a-z0-9]+)*\.)+[a-z]{'{'}2,{'}'}$"
				placeholder="eg: https://coollabs.io"
				required
				explainer={browser && window.location.hostname === 'demo.coolify.io'
					? "<span class='text-white font-bold'>You can use the predefined random url name or enter your own domain name.</span>"
					: "If you specify <span class='text-green-500 font-bold'>https</span>, the application will be accessible only over https. SSL certificate will be generated for you.<br>If you specify <span class='text-green-500 font-bold'>www</span>, the application will be redirected (302) from non-www and vice versa.<br><br>To modify the url, you must first stop the application.<br><br><span class='text-white font-bold'>You must set your DNS to point to the server IP in advance.</span>"}
			/>
			<div class="grid items-center pb-8 sm:grid-cols-2">
				<Setting
					dataTooltip="Must be stopped to modify."
					disabled={isRunning}
					isCenter={false}
					bind:setting={dualCerts}
					title="Generate SSL for www and non-www?"
					description="It will generate certificates for both www and non-www. <br>You need to have <span class='font-bold text-green-500'>both DNS entries</span> set in advance.<br><br>Useful if you expect to have visitors on both."
					on:click={() => !isRunning && changeSettings('dualCerts')}
				/>
			</div>
			{#if application.buildPack === 'python'}
				<div class="grid items-center sm:grid-cols-2">
					<label for="pythonModule" class="text-base font-bold text-stone-100">WSGI</label>
					<div class="custom-select-wrapper">
						<Select id="wsgi" items={wsgis} on:select={selectWSGI} value={application.pythonWSGI} />
					</div>
				</div>
				<Input
					label="Module"
					readonly={!$session.isAdmin}
					name="pythonModule"
					required
					bind:value={application.pythonModule}
					placeholder={application.pythonWSGI?.toLowerCase() !== 'gunicorn' ? 'main.py' : 'main'}
				/>

				{#if application.pythonWSGI?.toLowerCase() === 'gunicorn'}
					<Input
						label="Variable"
						readonly={!$session.isAdmin}
						name="pythonVariable"
						required
						bind:value={application.pythonVariable}
						placeholder="default: app"
					/>
				{/if}
			{/if}
			{#if !staticDeployments.includes(application.buildPack)}
				<Input
					label="Port"
					readonly={!$session.isAdmin}
					name="port"
					bind:value={application.port}
					placeholder={application.buildPack === 'python' ? '8000' : '3000'}
				/>
			{/if}

			{#if !notNodeDeployments.includes(application.buildPack)}
				<Input
					label="Install Command"
					readonly={!$session.isAdmin}
					name="installCommand"
					bind:value={application.installCommand}
					placeholder="default: yarn install"
				/>
				<Input
					label="Build Command"
					readonly={!$session.isAdmin}
					name="buildCommand"
					bind:value={application.buildCommand}
					placeholder="default: yarn build"
				/>

				<Input
					label="Start Command"
					readonly={!$session.isAdmin}
					name="startCommand"
					bind:value={application.startCommand}
					placeholder="default: yarn start"
				/>
			{/if}
			<Input
				label="Base Directory"
				explainer="Directory to use as the base for all commands.<br>Could be useful with <span class='text-green-500 font-bold'>monorepos</span>."
				readonly={!$session.isAdmin}
				name="baseDirectory"
				bind:value={application.baseDirectory}
				placeholder="default: /"
			/>

			{#if !notNodeDeployments.includes(application.buildPack)}
				<Input
					label="Publish Directory"
					explainer="Directory containing all the assets for deployment. <br> For example: <span class='text-green-500 font-bold'>dist</span>,<span class='text-green-500 font-bold'>_site</span> or <span class='text-green-500 font-bold'>public</span>."
					readonly={!$session.isAdmin}
					name="publishDirectory"
					bind:value={application.publishDirectory}
					placeholder=" default: /"
				/>
			{/if}
		</div>
	</form>
	<div class="flex space-x-1 pb-5 font-bold">
		<div class="title">Features</div>
	</div>
	<div class="px-2 pb-10 sm:px-10">
		<div class="grid items-center sm:grid-cols-2">
			<Setting
				isCenter={false}
				bind:setting={autodeploy}
				on:click={() => changeSettings('autodeploy')}
				title="Enable Automatic Deployment"
				description="Enable automatic deployment through webhooks."
			/>
		</div>
		<div class="grid items-center sm:grid-cols-2">
			<Setting
				isCenter={false}
				bind:setting={previews}
				on:click={() => changeSettings('previews')}
				title="Enable MR/PR Previews"
				description="Enable preview deployments from pull or merge requests."
			/>
		</div>
		<div class="grid items-center sm:grid-cols-2">
			<Setting
				isCenter={false}
				bind:setting={debug}
				on:click={() => changeSettings('debug')}
				title="Debug Logs"
				description="Enable debug logs during build phase.<br><span class='text-red-500 font-bold'>Sensitive information</span> could be visible and saved in logs."
			/>
		</div>
	</div>
</div>

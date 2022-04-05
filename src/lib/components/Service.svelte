<script>
	export let service;
	export let link;
</script>

<a href="/{link}/{service.id}" class="w-full p-2 no-underline sm:w-96">
	<div class="box-selection hover:bg-sky-600">
		<div class="truncate text-center text-xl font-bold">{service.name}</div>
		<div class="truncate text-center">{service.network}</div>
	</div>
</a>

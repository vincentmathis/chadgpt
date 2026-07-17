<script lang="ts">
import { onMount } from 'svelte';
import { goto } from '$app/navigation';

onMount(() => {
    goto('/chadgpt/', { replaceState: true });
});
</script>

<svelte:head>
    <title>ChadGPT</title>
</svelte:head>

<main>
    <p>Redirecting to ChadGPT...</p>
</main>

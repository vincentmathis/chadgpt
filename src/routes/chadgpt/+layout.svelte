<script lang="ts">
// biome-ignore-all lint/correctness/noUnusedImports: Svelte template usage
import '../layout.css';
const { children } = $props();
</script>

<svelte:head>
    <title>ChadGPT</title>
</svelte:head>

{@render children()}

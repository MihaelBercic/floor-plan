<script lang="ts">
    import CustomCanvas from "$lib/components/CustomCanvas.svelte";
    import Toolbar from "$lib/components/Toolbar.svelte";
</script>

<div id="container">
    <CustomCanvas/>
    <Toolbar/>
</div>

<style>
    #container {
        display: flex;
        position: relative;
        background-color: white;
        flex: 1;
    }

    #container canvas {
        flex: 1;
        background-color: rgba(0, 0, 0, .05);
        border: 1px solid rgba(0, 0, 0, .05);
    }
</style>
<script>
    import { onMount } from "svelte";
    import * as d3 from "d3";
    import TweedeKamer_perioden from '../TweedeKamer_perioden.json';

    onMount(async () => {
        const beginjaar = d3.group(TweedeKamer_perioden, d => d["Beginjaar"]);
        console.log(beginjaar);

        const eindjaar = d3.group(TweedeKamer_perioden, d => d['Eindjaar']);
        console.log(eindjaar);

    });
</script>

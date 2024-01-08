<script>
    import { onMount } from 'svelte';
    import * as d3 from 'd3';

    onMount(async () => {
        // fetch('../TweedeKamer_perioden.json')
        //     .then((response) => response.json())
        //     .then((data) => {
        //         console.log(data);
        //     });

        const data = d3.json('../TweedeKamer_perioden.json').then(function (d) {
            console.log(d);
        });

        // console.log(data)

        // const beginjaar = d3.group(TweedeKamer_perioden, d => d["Beginjaar"], d => d["Beginmaand"]);
        // console.log(beginjaar);

        // const eindjaar = d3.group(TweedeKamer_perioden, d => d['Eindjaar']);
        // console.log(eindjaar);
    });
</script>

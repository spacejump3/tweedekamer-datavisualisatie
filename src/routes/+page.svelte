<script>
    import Testcomponent from "../components/Testcomponent.svelte";
</script>

<body>
    <header>
        <h1>Zetelwisselingen door de jaren heen</h1>
        <p>
            In deze visualisatie kan je alle zetelperiodes in de tweede kamer
            zien. Wat we hiermee bedoelen is dat je exact kan zien hoeveel leden
            zijn begonnen, en weg zijn gegaan. Dit kan je zien in elke periode.
            Vanaf de tweede laag; de personen met een felle kleur zijn in die
            maand begonnen, en de personen met een lichtere kleur zijn gestopt.
            De kleuren zijn gebaseerd op geslacht, maar je kan ook zien in welke
            fractie elk lid zit.
        </p>
    </header>

    <Testcomponent />
</body>

<style>
    @import url("https://fonts.googleapis.com/css2?family=Roboto+Slab&display=swap");

    p,
    h1 {
        font-family: "Roboto Slab", serif;
    }
    p {
        width: 50rem;
        color: grey;
        margin-bottom: 3rem;
    }
</style>

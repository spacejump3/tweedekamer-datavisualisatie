<script>
    import Visualisatie from "../components/Visualisatie.svelte";
</script>

<body>
    <div>
        <header>
            <h1>Zetelwisselingen door de jaren heen</h1>
            <p>
                In deze visualisatie kan je alle zetelwisselingen in de tweede
                kamer zien. Wat we hiermee bedoelen is dat je exact kan zien
                hoeveel leden zijn begonnen, en weg zijn gegaan. Dit kan je zien
                in elke periode. Vanaf de tweede laag; de personen met een felle
                kleur zijn in die maand begonnen, en de personen met een
                lichtere kleur zijn gestopt. De kleuren zijn gebaseerd op
                geslacht, maar je kan ook zien in welke fractie elk lid zit.
            </p>
        </header>
        <section class="legendaOpacity">
            <h2>Kleuren</h2>
            <p>
                De transparantie van de kleur is gebaseerd of iemand is begonnen
                of gestopt
            </p>

            <div class="container">
                <div class="item">
                    <div></div>
                    <p>Persoon begonnen</p>
                </div>
                <div class="item">
                    <div></div>
                    <p>Persoon gestopt</p>
                </div>
            </div>
        </section>
    </div>

    <Visualisatie />
</body>

<style>
    @import url("https://fonts.googleapis.com/css2?family=Roboto+Slab&display=swap");

    p,
    h1,
    h2 {
        font-family: "Roboto Slab", serif;
    }
    header p {
        width: 40rem;
        color: grey;
        margin-bottom: 3rem;
    }

    body > div {
        display: flex;
        width: 1100px;
        justify-content: space-between;
        align-items: flex-start;
    }

    /* legenda opacity */
    .legendaOpacity {
        width: 15rem;
        text-align: right;
    }

    .legendaOpacity .container .item p {
        margin-top: 0.5rem;
        margin-bottom: 0.5rem;
    }

    .legendaOpacity p {
        color: grey;
        font-size: 0.9rem;
    }

    .legendaOpacity > h2 {
        margin: 0;
        margin-top: 2rem;
        font-size: 1.3rem;
    }

    .legendaOpacity > .container > .item {
        display: flex;
        align-items: center;
        justify-content: flex-end;
        gap: 1rem;
    }

    /* container of div and p */
    .legendaOpacity > .container > .item {
        display: flex;
    }

    .legendaOpacity > .container > .item > div {
        width: 1rem;
        height: 1rem;
    }

    .legendaOpacity > .container > div:nth-of-type(1) > div {
        background-color: rgb(33, 33, 33);
    }

    .legendaOpacity > .container > div:nth-of-type(2) > div {
        background-color: rgb(213, 213, 213);
    }
</style>

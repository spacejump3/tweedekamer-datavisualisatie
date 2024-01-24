<script>
    import Testcomponent from "../components/Testcomponent.svelte";
</script>

<body>
    <header>
    <h1>Tweedekamer wisselingen door de jaren heen</h1>
    <p>
        Visit <a href="https://kit.svelte.dev">kit.svelte.dev</a> to read the documentation
        Contrary to popular belief, Lorem Ipsum is not simply random text. It has
        roots in a piece of classical Latin literature from 45 BC, making it over
        2000 years old. Richard McClintock, a Latin professor at Hampden-Sydney College
        in Virginia, looked up one of the more obscure Latin words, consectetur,
        from a Lorem Ipsum passage, and going through the cites of the word in classical
        literature, discovered the undoubtable source. Lorem Ipsum comes from sections
        1.10.32 and 1.10.33 of "de Finibus Bonorum et Malorum" (The Extremes of Good
        and Evil) by Cicero, written in 45 BC. This book is a treatise on the theory
        
    </p>
    </header>

    <Testcomponent />
</body>

<style>
    @import url('https://fonts.googleapis.com/css2?family=Roboto+Slab&display=swap');

    header {
        display: flex;
        flex-direction: column;
        align-items: center;
        text-align: center;
    }

    p, h1 {
        font-family: 'Roboto Slab', serif;
    }
    p {
        width: 50rem;
        color: grey;
        margin-bottom: 4rem;
    }
</style>
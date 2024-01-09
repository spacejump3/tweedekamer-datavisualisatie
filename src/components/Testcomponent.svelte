<script>
    import { onMount } from 'svelte';
    import * as d3 from 'd3';

    onMount(async () => {
        const data = await d3.json('TweedeKamer_perioden.json');

        // Iterate through each object in the array
        for (var i = 0; i < data.length; i++) {
            if ('Naam (Uit personen))' in data[i]) {
                // Create a new key and assign the value of the old key
                data[i].Naam = data[i]['Naam (Uit personen))'];
                // Delete the old key
                delete data[i]['Naam (Uit personen))'];
            }
        }

        const groupedDataBeginjaar = d3.group(data, (d) => d['Beginjaar']);
        const groupedDataEindjaar = d3.group(data, (d) => d['Eindjaar']);

        // Convert the map to an array of objects and sort by Beginjaar
        const sortedResultBeginjaar = Array.from(
            groupedDataBeginjaar,
            ([beginYear, persons]) => {
                return {
                    Beginjaar: parseInt(beginYear, 10),
                    Persons: persons,
                };
            }
        ).sort((a, b) => a.Beginjaar - b.Beginjaar);

        // Convert the map to an array of objects and sort by Eindjjaar
        const sortedResultEindjaar = Array.from(
            groupedDataEindjaar,
            ([endYear, persons]) => {
                return {
                    Eindjaar: parseInt(endYear, 10),
                    Persons: persons,
                };
            }
        ).sort((a, b) => a.Eindjaar - b.Eindjaar);

        // console.log(sortedResultBeginjaar);
        // console.log(sortedResultEindjaar);

        // Process the sorted result and accumulate persons
        let accumulatedPersons = [];
        const finalResult = sortedResultBeginjaar.map(
            ({ Beginjaar, Persons }) => {
                // Include persons with starting year equal to the current year
                const personsFromCurrentYear = Persons.filter((person) => {
                    return parseInt(person['Beginjaar'], 10) === Beginjaar;
                });

                // Include persons from previous years if their ending year is equal to or later than the current year
                accumulatedPersons = accumulatedPersons.filter((person) => {
                    return parseInt(person['Eindjaar'], 10) >= Beginjaar;
                });

                // Add persons from the current year
                accumulatedPersons = accumulatedPersons.concat(
                    personsFromCurrentYear
                );

                return {
                    Beginjaar,
                    Persons: accumulatedPersons.slice(), // Copy the accumulatedPersons array to avoid reference issues
                };
            }
        );

        console.log(finalResult);
    });
</script>

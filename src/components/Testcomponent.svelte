<script>
    import { onMount } from 'svelte';
    import * as d3 from 'd3';

    onMount(async () => {
        const data = await d3.json('TweedeKamer_perioden.json');
        data.forEach((item) => (item.value = 1));

        // Iterate through each object in the array and rename the key
        for (var i = 0; i < data.length; i++) {
            if ('Naam (Uit personen))' in data[i]) {
                // Create a new key and assign the value of the old key
                data[i].Naam = data[i]['Naam (Uit personen))'];
                // Delete the old key
                delete data[i]['Naam (Uit personen))'];
            }
        }

        // Group by Beginjaar and Beginmaand
        const groupedData = d3.group(
            data,
            (d) => Math.floor((parseInt(d['Beginjaar'], 10) - 1815) / 16), // Group every 16 years starting from 1815
            (d) => d['Beginjaar'],
            (d) => d['Beginmaand']
        );

        // Convert the map to an array of objects and sort by Beginjaar
        const sortedResult = Array.from(groupedData, ([group, yearsData]) => {
            const sortedYearsData = Array.from(
                yearsData,
                ([beginYear, monthsData]) => {
                    const sortedMonthsData = Array.from(
                        monthsData,
                        ([beginMonth, persons]) => {
                            return {
                                Beginmaand: parseInt(beginMonth, 10),
                                children: persons,
                            };
                        }
                    ).sort((a, b) => a.Beginmaand - b.Beginmaand);

                    return {
                        Beginjaar: parseInt(beginYear, 10),
                        children: sortedMonthsData,
                    };
                }
            ).sort((a, b) => a.Beginjaar - b.Beginjaar);

            return {
                Group: group,
                children: sortedYearsData,
            };
        }).sort((a, b) => a.Group - b.Group);

        // Process the sorted result and accumulate persons
        let accumulatedPersons = [];
        const tweedeKamerData = sortedResult.map(({ Group, children }) => {
            // Process each year
            const processedYears = children.map(({ Beginjaar, children }) => {
                // Process each month
                const processedMonths = children.map(
                    ({ Beginmaand, children }) => {
                        // Include persons with starting year and month equal to the current year and month
                        const personsFromCurrentMonth = children.filter(
                            (person) => {
                                return (
                                    parseInt(person['Beginjaar'], 10) ===
                                        Beginjaar &&
                                    parseInt(person['Beginmaand'], 10) ===
                                        Beginmaand
                                );
                            }
                        );

                        // Include persons from previous years and months if their ending year is equal to or later than the current year and month
                        accumulatedPersons = accumulatedPersons.filter(
                            (person) => {
                                return (
                                    parseInt(person['Eindjaar'], 10) >
                                        Beginjaar ||
                                    (parseInt(person['Eindjaar'], 10) ===
                                        Beginjaar &&
                                        parseInt(person['Eindmaand'], 10) >=
                                            Beginmaand)
                                );
                            }
                        );

                        // Add persons from the current year and month
                        accumulatedPersons = accumulatedPersons.concat(
                            personsFromCurrentMonth
                        );

                        return {
                            Beginmaand,
                            children: accumulatedPersons.slice(), // Copy the accumulatedPersons array to avoid reference issues
                        };
                    }
                );

                return {
                    Beginjaar,
                    children: processedMonths,
                };
            });

            return {
                Group,
                children: processedYears,
            };
        });

        const finalResult = { year: 'root', children: tweedeKamerData };
        console.log(finalResult);
    });
</script>

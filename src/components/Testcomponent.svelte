<script>
    import { onMount } from "svelte";
    import * as d3 from "d3";

    onMount(async () => {
        const data = await d3.json("TweedeKamer_perioden.json");
        data.forEach((item) => (item.value = 1));

        // Iterate through each object in the array and rename the key
        for (var i = 0; i < data.length; i++) {
            if ("Naam (Uit personen))" in data[i]) {
                // Create a new key and assign the value of the old key
                data[i].Naam = data[i]["Naam (Uit personen))"];
                // Delete the old key
                delete data[i]["Naam (Uit personen))"];
            }
        }

        // Group by Beginjaar and Beginmaand
        const groupedData = d3.group(
            data,
            (d) => Math.floor((parseInt(d["Beginjaar"], 10) - 1815) / 16), // Group every 16 years starting from 1815
            (d) => d["Beginjaar"],
            (d) => d["Beginmaand"],
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
                        },
                    ).sort((a, b) => a.Beginmaand - b.Beginmaand);

                    return {
                        Beginjaar: parseInt(beginYear, 10),
                        children: sortedMonthsData,
                    };
                },
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
                                    parseInt(person["Beginjaar"], 10) ===
                                        Beginjaar &&
                                    parseInt(person["Beginmaand"], 10) ===
                                        Beginmaand
                                );
                            },
                        );

                        // Include persons from previous years and months if their ending year is equal to or later than the current year and month
                        accumulatedPersons = accumulatedPersons.filter(
                            (person) => {
                                return (
                                    parseInt(person["Eindjaar"], 10) >
                                        Beginjaar ||
                                    (parseInt(person["Eindjaar"], 10) ===
                                        Beginjaar &&
                                        parseInt(person["Eindmaand"], 10) >=
                                            Beginmaand)
                                );
                            },
                        );

                        // Add persons from the current year and month
                        accumulatedPersons = accumulatedPersons.concat(
                            personsFromCurrentMonth,
                        );

                        return {
                            Beginmaand,
                            children: accumulatedPersons.slice(), // Copy the accumulatedPersons array to avoid reference issues
                        };
                    },
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

        const finalResult = { year: "root", children: tweedeKamerData };
        // console.log(finalResult);

        const root = d3.hierarchy(finalResult);
        root.sum((d) => d.value);

        const layout = d3.treemap().size([1500, 800]).padding(1);
        // .tile(d3.treemapDice);

        layout(root);

        const xScale = d3.scaleLinear().range([0, 1500]);

        const yScale = d3.scaleLinear().range([0, 800]);

        const createTreemap = (data) => {
            const visualisation = d3
                .select("svg")
                .selectAll("g")
                .data(data)
                .join("g");

            // Create rectangles
            visualisation
                .append("rect")
                .attr("fill", "transparent")
                .attr("stroke", "black")
                .attr("x", function (d) {
                    return xScale(d.x0);
                })
                .attr("y", function (d) {
                    return yScale(d.y0);
                })
                .attr("width", function (d) {
                    return xScale(d.x1) - xScale(d.x0);
                })
                .attr("height", function (d) {
                    return yScale(d.y1) - yScale(d.y0);
                })
                .on("click", (e, d) => {
                    d3.selectAll("g").remove();
                    xScale.domain([d.x0, d.x1]);
                    yScale.domain([d.y0, d.y1]);
                    createTreemap(d.children);
                });

            // create label
            visualisation
                .append("text")
                .text((d) => Object.values(d.data)[0])
                .attr("x", (d) => xScale(d.x0) + 10)
                .attr("y", (d) => yScale(d.y0) + 20);
        };

        const onPageLoad = () => {
            xScale.domain([0, 1500]);
            yScale.domain([0, 800]);
            createTreemap(root.children);
        };

        onPageLoad();

    });
</script>

<section>
    <svg width="1500" height="1000"> </svg>
</section>

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
            (d) => Math.floor((parseInt(d["Beginjaar"], 10) - 1815) / 15), // Group every 15 years starting from 1815
            (d) => d["Beginjaar"],
            (d) => d["Beginmaand"],
        );

        function getMonthName(monthNumber) {
            const date = new Date();
            date.setMonth(monthNumber - 1);
            return date.toLocaleString("nl-NL", { month: "long" });
        }

        // Convert the map to an array of objects and sort by Beginjaar
        const sortedResult = Array.from(groupedData, ([group, yearsData]) => {
            const sortedYearsData = Array.from(
                yearsData,
                ([beginYear, monthsData]) => {
                    const sortedMonthsData = Array.from(
                        monthsData,
                        ([beginMonth, persons]) => {
                            return {
                                Beginmaand: getMonthName(
                                    parseInt(beginMonth, 10),
                                ),
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

        const nameGroupByYears = (children) => {
            let firstYear = children[0]["Beginjaar"];
            let lastYear = children[children.length - 1]["Beginjaar"];
            return `${firstYear} - ${lastYear}`;
        };

        // Process the sorted result and accumulate persons
        let accumulatedPersons = [];
        const tweedeKamerData = sortedResult.map(({ Group, children }) => {
            Group = nameGroupByYears(children);
            // Process each year
            const processedYears = children.map(({ Beginjaar, children }) => {
                // Process each month
                const processedMonths = children.map(
                    ({ Beginmaand, children }) => {
                        // Include persons with starting year and month equal to the current year and month
                        const personsFromCurrentMonth = children.filter(
                            (person) => {
                                person["Beginmaand"] = getMonthName(
                                    person["Beginmaand"],
                                );
                                return (
                                    parseInt(person["Beginjaar"], 10) ===
                                        Beginjaar &&
                                    person["Beginmaand"] === Beginmaand
                                );
                            },
                        );

                        // Include persons from previous years and months if their ending year is equal to or later than the current year and month
                        accumulatedPersons = accumulatedPersons.filter(
                            (person) => {
                                person["Eindmaand"] = getMonthName(
                                    person["Eindmaand"],
                                );
                                return (
                                    parseInt(person["Eindjaar"], 10) >
                                        Beginjaar ||
                                    (parseInt(person["Eindjaar"], 10) ===
                                        Beginjaar &&
                                        person["Eindmaand"] >= Beginmaand)
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

        const widthTreemap = 2000;
        const heightTreemap = 1000;

        const root = d3.hierarchy(finalResult);
        root.sum((d) => d.value);

        const layout = d3.treemap().size([widthTreemap, heightTreemap]);
        // .tile(d3.treemapDice);

        layout(root);

        const xScale = d3.scaleLinear().range([0, widthTreemap]);

        const yScale = d3.scaleLinear().range([0, heightTreemap]);

        let topLayer = 1;

        const updateTreemap = (d) => {
            xScale.domain([d.x0, d.x1]);
            yScale.domain([d.y0, d.y1]);
            let newData;
            if (d.depth === 0) {
                newData = d.children;
            } else {
                newData = d3.reverse(d.descendants());
                newData.pop();
            }
            topLayer = d3.min(newData, (item) => item.depth);
            createTreemap(newData);
        };

        const getLabels = (d) => {
            if (d.depth === 4) {
                return Object.values(d.data)[12];
            } else {
                return Object.values(d.data)[0];
            }
        };

        let breadcrumbsArr = [];

        const removeBreadcrumbs = (d) => {
            const findIndex = breadcrumbsArr.indexOf(d);
            breadcrumbsArr.splice(findIndex + 1);
            addBreadcrumb(breadcrumbsArr);
        };

        const createBreadcumbs = (d) => {
            const breadcrumName = getLabels(d);
            breadcrumbsArr.push({ name: breadcrumName, node: d });
            addBreadcrumb(breadcrumbsArr);
        };

        const addBreadcrumb = (breadcrumbArr) => {
            d3.select("#breadcrumps")
                .selectAll("li")
                .data(breadcrumbArr)
                .join(
                    (enter) => {
                        const select = enter.append("li");
                        select.append("a");
                        select.append("span");
                    },
                    (update) => update,
                    (exit) => exit.remove(),
                );

            d3.selectAll("ul a")
                .attr("href", "#")
                .text((d) => d["name"])
                .on("click", (e, d) => {
                    updateTreemap(d["node"]);
                    removeBreadcrumbs(d);
                });

            d3.selectAll("span").text(" >");

        };

        const createTreemap = (data) => {
            d3.select("svg")
                .selectAll("g")
                .data(data)
                .join(
                    (enter) => {
                        const group = enter.append("g");
                        group.append("rect");
                        group.append("text");
                    },
                    (update) => update,
                    (exit) => exit.remove(),
                );

            // Create rectangles
            d3.selectAll("g")
                .select("rect")
                .attr("opacity", "0.2")
                .attr("fill", "red")
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
                });

            // create label
            d3.selectAll("g")
                .select("text")
                .text((d) => {
                    // only show labels on the top layer
                    if (d.depth === topLayer) {
                        return getLabels(d);
                    }
                })
                .attr("x", (d) => xScale(d.x0) + 10)
                .attr("y", (d) => yScale(d.y0) + 20)
                .attr("fill", "black")
                .attr("opacity", "1");

            // making it clickable and create new treemap
            d3.selectAll("g").on("click", (e, d) => {
                // stop clicking when clicking on members
                if (d.depth !== 4) {
                    updateTreemap(d);
                    createBreadcumbs(d);
                } else {
                    return;
                }
            });
        };

        const onPageLoad = () => {
            xScale.domain([0, widthTreemap]);
            yScale.domain([0, heightTreemap]);
            createTreemap(root.children);
            createBreadcumbs(root);
        };

        onPageLoad();
    });
</script>

<section>
    <ul id="breadcrumps"></ul>
    <svg width="2300" height="1500"> </svg>
</section>

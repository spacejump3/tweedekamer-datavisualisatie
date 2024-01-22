<script>
    import { onMount } from "svelte";
    import * as d3 from "d3";

    onMount(async () => {
        const data = await d3.json("TweedeKamer_perioden.json");
        data.forEach((item) => (item.value = 1));

        // turn month number to a string name
        function getMonthName(monthNumber) {
            const date = new Date();
            date.setMonth(monthNumber - 1);
            return date.toLocaleString("nl-NL", { month: "long" });
        }

        // Iterate through each object in the array and rename the key
        for (var i = 0; i < data.length; i++) {
            if ("Naam (Uit personen))" in data[i]) {
                // Create a new key and assign the value of the old key
                data[i].Naam = data[i]["Naam (Uit personen))"];
                // Delete the old key
                delete data[i]["Naam (Uit personen))"];
            }

            if ("Eindmaand" in data[i]) {
                // Create a new key and assign the value of the old key
                data[i]["Eindmaand"] = getMonthName(data[i]["Eindmaand"]);
                // Delete the old key
                // delete data[i]["Naam (Uit personen))"];
            }

            if ("Beginmaand" in data[i]) {
                // Create a new key and assign the value of the old key
                data[i]["Beginmaand"] = getMonthName(data[i]["Beginmaand"]);
                // Delete the old key
                // delete data[i]["Naam (Uit personen))"];
            }
        }

        // Group by Beginjaar and Beginmaand
        const groupedDataBeginyear = d3.group(
            data,
            (d) => d["Beginjaar"],
            (d) => d["Beginmaand"],
        );

        // Group by Eindjaar en Eindmaand
        const groupedDataEndyear = d3.group(
            data,
            (d) => d["Eindjaar"],
            (d) => d["Eindmaand"],
        );

        // turn groupedDataEndyear internMap to an Array
        const sortEndYear = Array.from(
            groupedDataEndyear,
            ([endYear, monthsData]) => {
                const sortedMonthsData = Array.from(
                    monthsData,
                    ([endMonth, persons]) => {
                        return {
                            Beginmaand: endMonth,
                            children: persons,
                        };
                    },
                );
                return {
                    Beginjaar: parseInt(endYear, 10),
                    children: sortedMonthsData,
                };
            },
        );

        // Convert the map to an array of objects and sort by Beginjaar
        const sortBeginyear = Array.from(
            groupedDataBeginyear,
            ([beginYear, monthsData]) => {
                const sortedMonthsData = Array.from(
                    monthsData,
                    ([beginMonth, persons]) => {
                        return {
                            Beginmaand: beginMonth,
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

        // console.log("sortBeginyear: ");
        // console.log(sortBeginyear);

        // merge sortBeginyear and sortEndYear by year and month
        const test = sortBeginyear.map((beginjaar) => {
            sortEndYear.map((endYear) => {
                if (beginjaar["Beginjaar"] === endYear["Beginjaar"]) {
                    // check if sortEndYear contains new months that do not exist in sortBeginyear and put these in filter array
                    let filter = endYear["children"].filter((months2) => {
                        return !beginjaar["children"].some(
                            (month1) =>
                                months2["Beginmaand"] === month1["Beginmaand"],
                        );
                    });

                    filter = filter.flat();

                    beginjaar["children"].map((beginmaand) => {
                        endYear["children"].map((endMonth) => {
                            // if months are the same between sortEndYear and sortBeginyear, merge them together and reassign beginmaand["children"]
                            if (
                                endMonth["Beginmaand"] ===
                                beginmaand["Beginmaand"]
                            ) {
                                beginmaand["children"] = [
                                    ...beginmaand["children"],
                                    ...endMonth["children"],
                                ];
                            }
                        });

                        return {
                            beginmaand: beginmaand,
                            children: beginmaand["children"],
                        };
                    });
                    // reassign beginjaar["children"] with beginjaar["children"] and filter to add these extra months for each year
                    beginjaar["children"] = [
                        ...beginjaar["children"],
                        ...filter,
                    ];
                }
            });

            return {
                Beginjaar: beginjaar["Beginjaar"],
                children: beginjaar["children"],
            };
        });

        // find if sortEndYear has years that are not included in sortBeginyear and save them in new array
        let newEndYears = sortEndYear.filter(
            (obj1) =>
                !sortBeginyear.some(
                    (obj2) => obj1["Beginjaar"] === obj2["Beginjaar"],
                ),
        );

        // push these new years into array and sort them
        test.push(...newEndYears);
        test.sort((a, b) => {
            return a.Beginjaar - b.Beginjaar;
        });

        // create groups of 15 after all sortEndYear and sortBeginyear has been succefully merged together. creates an internMap
        const grouping2 = d3.group(
            test,
            (d) => Math.floor((parseInt(d["Beginjaar"], 10) - 1815) / 15), // Group every 15 years starting from 1815
        );

        // get the range of years for first treemap
        const nameGroupByYears = (children) => {
            let firstYear = children[0]["Beginjaar"];
            let lastYear = children[children.length - 1]["Beginjaar"];
            return `${firstYear} - ${lastYear}`;
        };

        // turn internMap of grouping into an array
        const tweedeKamerData = Array.from(grouping2, ([Group, children]) => {
            return {
                group: nameGroupByYears(children),
                children: children,
            };
        });

        // create object with a root to make it ready for a treemap
        const finalResult = { year: "root", children: tweedeKamerData };

        // 2000
        // 1000
        const widthTreemap = 1200;
        const heightTreemap = 700;

        // create hierarchy of data (node elements) and set value of what needs to be counted in treemap
        const root = d3.hierarchy(finalResult);
        // count all the d.value elements of 1
        root.sum((d) => d.value);

        // create treemap layout and set size
        const layout = d3.treemap().size([widthTreemap, heightTreemap]);
        // .tile(d3.treemapDice);

        layout(root);

        // create x and y scales
        const xScale = d3.scaleLinear().range([0, widthTreemap]);
        const yScale = d3.scaleLinear().range([0, heightTreemap]);

        // the top layer of the treemap that can be clicked on
        let topLayer = 1;

        // get new data when new treemap is about to be created
        const updateTreemap = (d) => {
            // change scale based on new dimensions
            xScale.domain([d.x0, d.x1]);
            yScale.domain([d.y0, d.y1]);
            let newData;
            // update new data
            // if at top layer (root / 0) only show children. Else also get descendants.
            if (d.depth === 0) {
                newData = d.children;
            } else {
                newData = d3.reverse(d.descendants());
                newData.pop();
            }
            // update top layer of treemap that can be clicked on
            topLayer = d3.min(newData, (item) => item.depth);
            // create treemap with new data
            createTreemap(newData);
        };

        // get names of the labels
        const getLabels = (d) => {
            // bottom layer (4 / members) is at index 12. The rest at 0
            if (d.depth === 4) {
                return Object.values(d.data)[15];
            } else {
                return Object.values(d.data)[0];
            }
        };

        let breadcrumbsArr = [];

        const removeBreadcrumbs = (d) => {
            // remove excess breadcrumbs in array
            const findIndex = breadcrumbsArr.indexOf(d);
            breadcrumbsArr.splice(findIndex + 1);
            // create new breadcrumb on page with updated array
            createBreadcrumbs(breadcrumbsArr);
        };

        const updateBreadcrumbs = (d) => {
            // get name of the new breadcrumb
            const breadcrumName = getLabels(d);
            // push object to array containing name of breadcrumb and Node / data
            breadcrumbsArr.push({ name: breadcrumName, node: d });
            // create new breadcrumb on page with updated array
            createBreadcrumbs(breadcrumbsArr);
        };

        const createBreadcrumbs = (breadcrumbArr) => {
            // make breadcrumbs appear on page
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
                    // update treemap after clicking on a breadcrumb
                    updateTreemap(d["node"]);
                    removeBreadcrumbs(d);
                });

            d3.selectAll("span").text(" >");
        };

        const createTreemap = (data) => {
            // make treemap appear on page
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
                .attr("opacity", (d) => {
                    if (
                        d.depth === 4 &&
                        d.data["Eindjaar"] ==
                            d.parent.parent.data["Beginjaar"] &&
                        d.data["Eindmaand"] === d.parent.data["Beginmaand"]
                    ) {
                        return "0.1";
                    } else {
                        return "0.3";
                    }
                })
                .attr("fill", (d) => {
                    if (d.depth === 4 || d.depth === 1) {
                        return "red";
                    } else {
                        return "transparent";
                    }
                })

                .attr("stroke", "black")
                .attr("stroke-width", (d) => (d.depth === topLayer ? "3" : "1"))
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
                    updateBreadcrumbs(d);
                } else {
                    return;
                }
            });

            d3.selectAll("rect")
                .on("mouseover", (e, d) => {
                    // console.log(e.currentTarget)
                    d3.selectAll(this).style("transform", "scale(2)");

                    if (d.depth === 4) {
                        d3.select("#tooltip")
                            .style("opacity", 1)
                            .select("p:nth-of-type(1)")
                            .text(`Naam: ${d.data["Voornamen"]} ${d.data["Naam"]}`);

                        d3.select("#tooltip")
                            .select("p:nth-of-type(2)")
                            .text(
                                `Begindatum: ${d.data["Beginjaar"]} ${d.data["Beginmaand"]}`,
                            );

                        d3.select("#tooltip")
                            .select("p:nth-of-type(3)")
                            .text(
                                `Einddatum: ${d.data["Eindjaar"]} ${d.data["Eindmaand"]}`,
                            );

                        d3.select("#tooltip")
                            .select("p:nth-of-type(4)")
                            .text(`Fractie: ${d.data["Fractie"]}`);
                    }
                })
                .on("mousemove", (e, d) => {
                    if (d.depth === 4) {
                        d3.select("#tooltip")
                            .style("opacity", 1)
                            .style("position", "absolute")
                            .style("left", e.pageX + 15 + "px")
                            .style("top", e.pageY + 15 + "px");
                    }
                })

                .on("mouseout", (e, d) => {
                    d3.select("#tooltip").style("opacity", "0");
                });
        };

        d3.select("#fractieFilter").on("change", () => {
            // turn on filter

            const isChecked = d3.select("#fractieFilter").property("checked");
            if (isChecked) {
                const fracties = d3.group(data, (d) => d["Fractie"]);

                const colorScale = d3
                    .scaleOrdinal()
                    .domain(fracties)
                    .range(d3.schemePaired);

                d3.selectAll("rect").attr("fill", (d) =>
                    d.depth === 4
                        ? colorScale(d.data["Fractie"])
                        : "transparent",
                );
            } else {
                d3.selectAll("rect").attr("fill", (d) => {
                    if (d.depth === 4 || d.depth === 1) {
                        return "red";
                    } else {
                        return "transparent";
                    }
                });
            }
        });

        const checkFilter = () => {

        }
        const filterFunction = () => {

        }

        const onPageLoad = () => {
            xScale.domain([0, widthTreemap]);
            yScale.domain([0, heightTreemap]);
            updateTreemap(root);
            updateBreadcrumbs(root);
        };

        onPageLoad();
    });
</script>

<ul id="breadcrumps"></ul>
<svg width="2300" height="1500"> </svg>
<div id="tooltip">
    <p></p>
    <p></p>
    <p></p>
    <p></p>
</div>
<input id="fractieFilter" type="checkbox" />Fracties

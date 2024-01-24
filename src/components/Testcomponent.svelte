<script>
    import { onMount } from "svelte";
    import * as d3 from "d3";

    const widthTreemap = 1100;
    const heightTreemap = 600;

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
            }

            if ("Beginmaand" in data[i]) {
                // Create a new key and assign the value of the old key
                data[i]["Beginmaand"] = getMonthName(data[i]["Beginmaand"]);
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

        // merge sortBeginyear and sortEndYear by year and month
        const mergedData = sortBeginyear.map((beginjaar) => {
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
        mergedData.push(...newEndYears);
        mergedData.sort((a, b) => {
            return a.Beginjaar - b.Beginjaar;
        });

        // remove NaN
        mergedData.pop();

        // create groups of 15 after all sortEndYear and sortBeginyear has been succefully merged together. creates an internMap
        const mergedDataGroup = d3.group(
            mergedData,
            (d) => Math.floor((parseInt(d["Beginjaar"], 10) - 1815) / 15), // Group every 15 years starting from 1815
        );

        // get the range of years for first treemap
        const nameGroupByYears = (children) => {
            let firstYear = children[0]["Beginjaar"];
            let lastYear = children[children.length - 1]["Beginjaar"];
            return `${firstYear} - ${lastYear}`;
        };

        // turn internMap of grouping into an array
        const tweedeKamerData = Array.from(
            mergedDataGroup,
            ([Group, children]) => {
                return {
                    group: nameGroupByYears(children),
                    children: children,
                };
            },
        );

        // create object with a root to make it ready for a treemap
        const finalResult = { year: "Root", children: tweedeKamerData };

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
        let colorScale;

        // the top layer of the treemap that can be clicked on
        let topLayer = 1;
        let newData;

        // get new data when new treemap is about to be created
        const updateTreemap = (d) => {
            // change scale based on new dimensions
            xScale.domain([d.x0, d.x1]);
            yScale.domain([d.y0, d.y1]);
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
            // check which filter is on and apply it
            const filterType = checkFilter();
            filterFunction(filterType);
            createLegenda(newData);
        };

        const createLegenda = (data) => {
            const filterType = checkFilter();
            data = data.filter((item) => item.depth === 4);
            const groupFracties = d3.group(data, (d) => d.data[filterType]);

            d3.select("section")
                .style("display", "flex")
                .style("flex-direction", "column");

            d3.select("#legenda")
                .selectAll("#legenda > div")
                .data(groupFracties)
                .join((enter) => {
                    const group = enter.append("div");
                    group.append("div");
                    group.append("p");
                });

            d3.select("#legenda")
                .selectAll("#legenda > div")
                .style("display", "flex")
                .style("align-items", "center")
                .style("margin-bottom", "0.5rem")
                .style("order", (d) => (d[0] === "overige" ? "3" : ""));

            d3.select("#legenda")
                .selectAll("#legenda > div")
                .select("p")
                .text((d) => d[0])
                .style("margin", "0")
                .style("margin-left", "0.5rem")
                .style('font-family', 'Roboto Slab')

            d3.select("#legenda")
                .selectAll("#legenda > div")
                .select("div")
                .style("width", "1rem")
                .style("height", "1rem")
                .style("background-color", (d) => colorScale(d[0]));
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
                .style("text-decoration", "none")
                .style("padding", "0.4rem")
                .style("color", "blue")
                .style("font-size", "17px")
                .style("font-family", 'Roboto Slab')
                .on("click", (e, d) => {
                    // update treemap after clicking on a breadcrumb
                    updateTreemap(d["node"]);
                    removeBreadcrumbs(d);
                });

            d3.selectAll("span").text(" >");
        };

        const createTreemap = (data) => {
            // make treemap appear on page
            d3.select("#treemap")
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
            d3.selectAll("#treemap g")
                .select("rect")
                .attr("opacity", (d) => {
                    if (
                        d.depth === 4 &&
                        d.data["Eindjaar"] ==
                            d.parent.parent.data["Beginjaar"] &&
                        d.data["Eindmaand"] === d.parent.data["Beginmaand"]
                    ) {
                        return "0.2";
                    } else {
                        return "0.5";
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
            d3.selectAll("#treemap g")
                .select("text")
                .text((d) => {
                    // only show labels on the top layer
                    if (d.depth !== 4 && d.depth === topLayer) {
                        return getLabels(d);
                    }
                })
                .attr("x", (d) => xScale(d.x0) + 10)
                .attr("y", (d) => yScale(d.y0) + 20)
                .attr("fill", "black")
                .attr("opacity", "1")
                .attr("filter", "url(#solid)")
                .attr('font-size', '13px')
                .attr('font-family', 'Roboto Slab')

            // making it clickable and create new treemap
            d3.selectAll("#treemap g").on("click", (e, d) => {
                // stop clicking when clicking on members
                if (d.depth !== 4) {
                    updateTreemap(d);
                    updateBreadcrumbs(d);
                } else {
                    return;
                }
            });

            d3.selectAll("#treemap rect")
                .on("mouseover", (e, d) => {
                    if (d.depth === 4) {
                        d3.select("#tooltip")
                            .style("opacity", 1)
                            .select("#naam")
                            .text(`${d.data["Voornamen"]} ${d.data["Naam"]}`);

                        d3.select("#tooltip")
                            .select("#levensduur")
                            .text(
                                `(${d.data["Geboortejaar"]} - ${d.data["Sterfjaar"]})`,
                            );

                        d3.select("#tooltip")
                            .select("#geslacht")
                            .text(`${d.data["Geslacht"]}`);

                        d3.select("#tooltip")
                            .select("#begindatum")
                            .text(
                                `${d.data["Beginmaand"]} ${d.data["Beginjaar"]}`,
                            );

                        d3.select("#tooltip")
                            .select("#einddatum")
                            .text(
                                `${d.data["Eindmaand"]} ${d.data["Eindjaar"]}`,
                            );

                        d3.select("#tooltip")
                            .select("#fractie")
                            .text(`${d.data["Fractie"]}`);
                    }
                })
                .on("mousemove", (e, d) => {
                    if (d.depth === 4) {
                        d3.select("#tooltip")
                            .style("opacity", 1)
                            .style("position", "absolute")
                            .style("left", e.pageX + 10 + "px")
                            .style("top", e.pageY + 10 + "px");
                    }
                })

                .on("mouseout", (e, d) => {
                    d3.select("#tooltip").style("opacity", "0");
                });
        };

        d3.selectAll("[name=filter]").on("change", () => {
            // turn on filter
            const filterType = checkFilter();
            filterFunction(filterType);
            // create new legenda
            createLegenda(newData);
        });

        const checkFilter = () => {
            // check which filter is checked
            if (d3.select("#fractieFilter").property("checked")) {
                return "fractieFilter";
            } else {
                return "Geslacht";
            }
        };

        const filterFunction = (filterType) => {
            // create group of unique values of filterType
            const valuesFilter = d3.group(data, (d) => d[filterType]);

            // create array with colors with built-in colorschemes
            let extendedColors = [...d3.schemePaired, ...d3.schemeTableau10];

            // create colorScale
            colorScale = d3
                .scaleOrdinal()
                .domain(valuesFilter)
                .range(extendedColors);

            d3.selectAll("#treemap rect").attr("fill", (d) => {
                if (d.depth === 4) {
                    // use filters on members rectangles
                    return colorScale(d.data[filterType]);
                } else if (d.depth === 1) {
                    // if layer is first treemap (rnage of years) make all rectangles red
                    return "rgb(100, 100, 224)";
                } else {
                    // all other layers are transparent
                    return "transparent";
                }
            });
        };

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
<svg width={widthTreemap + 200} height={heightTreemap + 30} id="treemap">
    <defs>
        <filter x="0" y="0" width="1" height="1" id="solid">
          <feFlood flood-color="white" result="bg" />
          <feMerge>
            <feMergeNode in="bg"/>
            <feMergeNode in="SourceGraphic"/>
          </feMerge>
        </filter>
      </defs>
     </svg>
<div id="tooltip">
    <div>
        <p>Naam:</p>
        <p id="naam"></p>
        <p id="levensduur"></p>
    </div>
    <div>
        <p>Geslacht:</p>
        <p id="geslacht"></p>
    </div>
    <div>
        <p>Begindatum:</p>
        <p id="begindatum"></p>
    </div>
    <div>
        <p>Einddatum:</p>
        <p id="einddatum"></p>
    </div>
    <div>
        <p>Fractie:</p>
        <p id="fractie"></p>
    </div>
</div>

<div id="container">
    <section id="legenda">
        <h2>Legenda</h2>
    </section>

    <section id="filter">
        <h2>Filter op</h2>
        <div>
        <input
            type="radio"
            name="filter"
            id="geslachtFilter"
            value="fractie"
            checked
        />
        <label for="geslachtFilter">Geslacht</label>
        <input
            type="radio"
            name="filter"
            id="fractieFilter"
            value="fractie"
        /><label for="fractieFilter">Fractie</label>
        </div>
    </section>
</div>

<style>
    @import url('https://fonts.googleapis.com/css2?family=Roboto+Slab:wght@400;700&display=swap');

    p,
    label,
    h2 {
        font-family: 'Roboto Slab', serif;
    }

    h2 {
        margin: 0;
        margin-bottom: 1rem;
    }

    /* container legenda and filter */ 
    #container {
        display: grid;
        grid-template-rows: 1fr;
        grid-template-columns: 1fr 1fr 1fr;
        width: 1100px;
        margin-bottom: 5rem;
        
    }
    /* Filters */
    #filter {
        justify-self: center;
        text-align: center;
    }

    #filter div {
        display: flex;
        align-items: flex-start;
    }

    #filter label {
        background-color: rgb(247, 247, 247);
        border: solid rgb(100, 100, 224);
        padding: 1rem;
    }

    #filter input {
        display: none;
    }

    #filter input:checked + label {
        background-color: rgb(100, 100, 224);
        color: white;
    }

    /* Breadcrumbs */
    ul {
        padding: 0;
        list-style: none;
        display: flex;
    }
    /* Tooltip */
    #levensduur {
        font-size: 0.8rem;
        text-align: center;
    }

    #tooltip {
        position: absolute;
        left: -20%;
        top: -20%;
        background-color: rgb(100, 100, 224);
        padding: 1rem;
        border-radius: 0.5rem;
    }

    #tooltip div {
        display: flex;
        align-items: baseline;
    }

    #tooltip p {
        margin: 0;
        margin-bottom: 0.5rem;
        color: white;
    }

    #tooltip div p:first-child {
        margin-right: 0.5rem;
    }

    #tooltip div p:nth-child(2) {
        font-weight: bold;
    }
</style>

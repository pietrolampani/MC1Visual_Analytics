<template>
  <div>
    <div class="row">
      <div class="col">
        <form @submit.prevent="search">
          <div class="mb-3 form-row-container">
            <label for="dropdownId" class="form-label">Select an id</label>
            <Multiselect
              v-model="selectedIds"
              :options="uniqueIds"
              :multiple="true"
              :close-on-select="false"
              :clear-on-select="false"
              :preserve-search="true"
              :preselect-first="true"
              id="dropdownId"
            >
            </Multiselect>
          </div>
          <div class="mb-3 form-row-container">
            <label for="dropdownCountry" class="form-label"
              >Select a country</label
            >
            <Multiselect
              v-model="selectedCountries"
              :options="uniqueCountries"
              :multiple="true"
              :close-on-select="false"
              :clear-on-select="false"
              :preserve-search="true"
              :preselect-first="true"
              id="dropdownCountry"
            >
            </Multiselect>
          </div>
          <div class="row mt-3">
            <div class="col submit-container">
              <button type="submit" class="btn btn-primary">Submit</button>
            </div>
          </div>
        </form>
      </div>
    </div>

    <div class="row mt-3">
      <div class="col">
        <Multiselect
          v-model="selectedCountries"
          :options="uniqueCountries"
          :multiple="true"
          :close-on-select="false"
          :clear-on-select="false"
          :preserve-search="true"
          :preselect-first="true"
        ></Multiselect>
      </div>
      fix
    </div>

    <div class="row mt-3">
      <div class="col submit-container">
        <button type="submit" class="btn btn-primary">Submit</button>
      </div>
    </div>

    <div class="row mt-3">
      <div class="col">
        <div class="graph-container">
          <div id="my_dataviz_svg"></div>
          <div id="legends">
            <div id="my_node_legend">
              <h3>Legend for Nodes</h3>
            </div>
            <div id="my_link_legend">
              <h3>Legend for Links</h3>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import * as d3 from "d3";
import Multiselect from "vue-multiselect";

const colorMapNodes = {
  vessel: "#0000FF",
  political_organization: "#FF0000",
  person: "#800080",
  organization: "#FFA500",
  movement: "#FFFF00",
  location: "#00FFFF",
  event: "#008000",
  company: "#A52A2A",
  unspecified: "gray",
};
const colorMapLinks = {
  ownership: "#FFD700",
  partnership: "#98FB98",
  family_relationship: "#FFA07A",
  membership: "#87CEEB",
};

export default {
  components: {
    Multiselect,
  },
  data() {
    return {
      originalNodes: [], // Aggiungi un array per i nodi originali
      originalLinks: [], // Aggiungi un array per i link originali
      uniqueCountries: [],
      uniqueIds: [],
      selectedIds: [],
      selectedCountries: [],
      nodeFilters: {
        // Aggiungi un oggetto per i filtri dei nodi
        vessel: true,
        political_organization: true,
        person: true,
        organization: true,
        movement: true,
        location: true,
        event: true,
        company: true,
        unspecified: true,
      },
      linkFilters: {
        ownership: true,
        partnership: true,
        family_relationship: true,
        membership: true,
      },
    };
  },
  mounted() {
    //esportazione dei metodi
    this.loadOriginalData(); // Carica i dati originali al primo caricamento
    this.createLegend();
    this.renderGraph();
  },
  methods: {
    loadOriginalData() {
      fetch("/MC1.json")
        .then((res) => res.json())
        .then((data) => {
          this.originalNodes = data.nodes;
          this.originalLinks = data.links;
          this.uniqueIds = data.nodes.map((node) => node.id);
          this.uniqueCountries = Array.from(
            new Set(this.originalNodes.map((node) => node.country))
          );
          this.uniqueCountries.sort();
        })
        .catch((error) => console.log(error));
    },
    async search() {
      try {
        //Recupera gli attributi selezionati
        const selectedId = this.selectedIds.map((id) => id);
        const selectedCountry = this.selectedCountries.map((c) => c);
        let filteredNodes = this.originalNodes.map((n) => ({ ...n }));
        let filteredLinks = this.originalLinks.map((l) => ({ ...l }));

        const filterLinks = (allNodes, country) => {
          return filteredLinks.filter((link) => {
            const hasTarget = selectedId.includes(link.target);
            const hasSource = selectedId.includes(link.source);
            const uniqueNodes = filteredNodes.map((n) => n.id);

            if (hasSource || hasTarget) {
              const linkedNode = allNodes.find((n) =>
                hasTarget ? n.id === link.source : n.id === link.target
              );
              if (country)
                if (!selectedCountry.includes(linkedNode.country)) return false;
              if (linkedNode && !uniqueNodes.includes({ ...linkedNode }.id)) {
                filteredNodes.push({ ...linkedNode });
                uniqueNodes.push({ ...linkedNode }.id);
                return true;
              }
            }
            return false;
          });
        };

        // Se almeno un id è selezionato...
        if (selectedId.length) {
          filteredNodes = filteredNodes.filter((n) =>
            selectedId.includes(n.id)
          );
          // ...e almeno una country è selezionata,
          // filtra solo i nodi collegati a quelli selezionati nelle country selezionate;
          // altrimenti, seleziona tutti i nodi collegati
          filteredLinks = filterLinks(
            this.originalNodes,
            !!selectedCountry.length
          ).map((l) => ({ ...l }));
        } else if (selectedCountry.length) {
          // Se almeno una country è selezionata...
          filteredNodes = filteredNodes
            .filter((n) => selectedCountry.includes(n.country))
            .map((n) => ({ ...n }));
          // ...trova i vari collegamenti tra i nodi della country
          filteredLinks = filterLinks(filteredNodes).map((l) => ({ ...l }));
        }

        // Pulisce il grafico
        d3.select("#my_dataviz_svg").selectAll("*").remove();

        // Renderizza il grafico con i nodi e i link filtrati
        this.renderGraph(filteredNodes, filteredLinks);
      } catch (error) {
        console.error("[GraphChart] Error on search(): ", error);
        alert("Error performing search:", error);
      }
    },

    //metodo che costruisce il grafico
    async renderGraph(nodes = this.originalNodes, links = this.originalLinks) {
      try {
        // Se nodes e links sono vuoti o undefined, carica i valori predefiniti da "/MC1.json"
        // if (!(nodes)) {
        //     nodes = this.originalNodes;
        // } else if (!(links)) {
        // 		links = this.originalLinks
        // }

        // Trova il nodo con il maggior numero di link

        const nodeLinksCount = {};
        links.forEach((link) => {
          nodeLinksCount[link.source] = (nodeLinksCount[link.source] || 0) + 1;
          nodeLinksCount[link.target] = (nodeLinksCount[link.target] || 0) + 1;
        });

        const maxLinkCountNode = Object.keys(nodeLinksCount).reduce(
          (a, b) => (nodeLinksCount[a] > nodeLinksCount[b] ? a : b),
          ""
        );

        // Seleziona tutti i link collegati al nodo con il maggior numero di link
        const filteredLinks = links.filter(
          (link) =>
            link.source === maxLinkCountNode || link.target === maxLinkCountNode
        );

        // Seleziona tutti i nodi connessi ai link selezionati
        const connectedNodes = new Set();
        filteredLinks.forEach((link) => {
          connectedNodes.add(link.source);
          connectedNodes.add(link.target);
        });

        // Inizializzazione del container SVG
        const width = 7000;
        const height = 7000;
        const svg = d3
          .create("svg")
          .attr("viewBox", [0, 0, width, height])
          .attr("width", "700px")
          .attr("height", "700px");

        // Inizializzazione dei nodi
        const node = svg
          .selectAll("circle")
          .data(nodes)
          .enter()
          .append("circle")
          .attr("r", 10)
          .attr("fill", (d) => colorMapNodes[d.type] || "grey") // Usa la mappa dei colori per i nodi
          .attr("cx", (d) => d.x)
          .attr("cy", (d) => d.y);

        // Inizializzazione dei link
        const link = svg
          .selectAll("line")
          .data(filteredLinks)
          .enter()
          .append("line")
          .attr("stroke", (d) => colorMapLinks[d.type] || "grey") // Usa la mappa dei colori per i link
          .attr("x1", (d) => d.source.x)
          .attr("y1", (d) => d.source.y)
          .attr("x2", (d) => d.target.x)
          .attr("y2", (d) => d.target.y);

        // Inizializzazione del tooltip
        const Tooltip = d3
          .select("#my_dataviz_svg")
          .append("div")
          .style("opacity", 0)
          .attr("class", "tooltip")
          .style("background-color", "white")
          .style("color", "black")
          .style("border", "solid")
          .style("border-width", "2px")
          .style("border-radius", "5px")
          .style("padding", "10px");

        // Funzione che mostra il tooltip al passaggio del mouse su un nodo
        const mouseover = function (event, d) {
          Tooltip.style("opacity", 1)
            .html(
              `
                        Name: ${d.id ? d.id : "N/A"}<br>
                        Country: ${d.country ? d.country : "N/A"}<br>
                        Type: ${d.type ? d.type : "N/A"}
                    `
            )
            .style("left", event.pageX + "px")
            .style("top", event.pageY + "px");
        };

        // Funzione che nasconde il tooltip se si ci si sposta col mouse da un nodo
        const mouseleave = function () {
          Tooltip.transition().duration(200).style("opacity", 0);
        };

        // Funzione che gestisce lo zoom se si clicca su un nodo
        const clickNode = function (event, d) {
          // Rimuove l'evidenziazione dai nodi precedentemente cliccati
          svg.selectAll(".highlight").remove();

          // Evidenzia il nodo cliccato con un cerchio verde fluo
          d3.select(this)
            .append("circle")
            .attr("class", "highlight")
            .attr("r", 45)
            .attr("fill", "lime")
            .attr("stroke", "black")
            .attr("stroke-width", 2)
            .attr("pointer-events", "none");

          // Zoom sul nodo cliccato
          const { x, y } = d;
          svg
            .transition()
            .duration(750)
            .call(
              zoom.transform,
              d3.zoomIdentity.translate(width / 2 - x, height / 2 - y).scale(64)
            );

          // Mostra il tooltip
          Tooltip.style("opacity", 1)
            .html(
              `
                        Name: ${d.id ? d.id : "N/A"}<br>
                        Country: ${d.country ? d.country : "N/A"}<br>
                        Type: ${d.type ? d.type : "N/A"}
                    `
            )
            .style("left", event.pageX + "px")
            .style("top", event.pageY + "px");

          // Esclude l'evento mouseleave
          Tooltip.on("mouseleave", null);
        };

        // Assegnazione delle funzioni ai nodi
        node
          .on("mouseover", mouseover)
          .on("mouseleave", mouseleave)
          .on("click", clickNode);

        // Inizializzazione delle forze e del comportamento dello zoom
        const simulation = d3
          .forceSimulation(nodes)
          .force(
            "link",
            d3.forceLink(links).id((d) => d.id)
          )
          .force("charge", d3.forceManyBody().strength(-100))
          .force("center", d3.forceCenter(width / 2, height / 2));

        simulation.on("tick", () => {
          link
            .attr("x1", (d) => d.source.x)
            .attr("y1", (d) => d.source.y)
            .attr("x2", (d) => d.target.x)
            .attr("y2", (d) => d.target.y);

          node.attr("cx", (d) => d.x).attr("cy", (d) => d.y);
        });

        const zoomed = (event) => {
          const { transform } = event;
          link.attr("transform", transform);
          node.attr("transform", transform);
        };

        //fuznione di szoom
        const zoom = d3.zoom().scaleExtent([1, 128]).on("zoom", zoomed);

        svg.call(zoom);

        // Aggiungi il grafico al div
        document.getElementById("my_dataviz_svg").appendChild(svg.node());
      } catch (error) {
        console.error("Error rendering graph:", error);
      }
    },
    async loadOptions() {
      try {
        //recupera i nomi dei nodi
        const response = await fetch("/MC1.json");
        const mc1 = await response.json();

        const dropdownId = document.getElementById("dropdownId");
        // const dropdownCountry = document.getElementById('dropdownCountry');
        const nodes = mc1.nodes;

        console.log("Number of nodes:", nodes.length); // Debug
        //carica i nodi nella select delle option
        if (nodes) {
          nodes.forEach((node) => {
            console.log("Country:", node.country); // Debug
            const optionId = document.createElement("option");
            optionId.value = node.id;
            optionId.textContent = node.id;
            dropdownId.appendChild(optionId);

            // controllo di vuotezza del campo country
            // if (node.country && node.country.trim() !== '') {
            //     const optionCountry = document.createElement('option');
            //     optionCountry.value = node.country;
            //     optionCountry.textContent = node.country;
            //     dropdownCountry.appendChild(optionCountry);
            // }
          });
        }
      } catch (error) {
        console.error("Error loading options:", error);
      }
    },
    createLegend() {
      try {
        const nodeLegendDiv = document.getElementById("my_node_legend");
        Object.keys(colorMapNodes).forEach((type) => {
          const nodeLegendItem = document.createElement("div");
          nodeLegendItem.style.display = "flex";
          nodeLegendItem.style.alignItems = "center";

          const nodeLegendColor = document.createElement("div");
          nodeLegendColor.style.width = "20px";
          nodeLegendColor.style.height = "20px";
          nodeLegendColor.style.backgroundColor = colorMapNodes[type];
          nodeLegendColor.style.marginRight = "5px";

          const nodeLegendLabel = document.createElement("label");
          nodeLegendLabel.textContent = type;

          const nodeLegendCheckbox = document.createElement("input");
          nodeLegendCheckbox.type = "checkbox";
          nodeLegendCheckbox.checked = true;
          nodeLegendCheckbox.value = type;
          nodeLegendCheckbox.addEventListener("change", () =>
            this.handleNodeFilterChange(type)
          );

          nodeLegendItem.appendChild(nodeLegendColor);
          nodeLegendItem.appendChild(nodeLegendLabel);
          nodeLegendItem.appendChild(nodeLegendCheckbox);
          nodeLegendDiv.appendChild(nodeLegendItem);
        });

        const linkLegendDiv = document.getElementById("my_link_legend");
        Object.keys(colorMapLinks).forEach((type) => {
          const linkLegendItem = document.createElement("div");
          linkLegendItem.style.display = "flex";
          linkLegendItem.style.alignItems = "center";

          const linkLegendColor = document.createElement("div");
          linkLegendColor.style.width = "20px";
          linkLegendColor.style.height = "20px";
          linkLegendColor.style.backgroundColor = colorMapLinks[type];
          linkLegendColor.style.marginRight = "5px";

          const linkLegendLabel = document.createElement("label");
          linkLegendLabel.textContent = type;

          const LinkLegendCheckbox = document.createElement("input");
          LinkLegendCheckbox.type = "checkbox";
          LinkLegendCheckbox.checked = true;
          LinkLegendCheckbox.value = type;
          LinkLegendCheckbox.addEventListener("change", () =>
            this.handleLinkFilterChange(type)
          );

          linkLegendItem.appendChild(linkLegendColor);
          linkLegendItem.appendChild(linkLegendLabel);
          linkLegendItem.appendChild(LinkLegendCheckbox);
          linkLegendDiv.appendChild(linkLegendItem);
        });
      } catch (error) {
        console.error("Error creating legend:", error);
      }
    },
    handleNodeFilterChange(type) {
      try {
        this.nodeFilters[type] = !this.nodeFilters[type];

        d3.selectAll("circle")
          .filter((d) => {
            if (d === this.currentNode) {
              return false;
            }
            if (type === "unspecified") {
              return !d.type;
            } else {
              return d.type === type;
            }
          })
          .style("visibility", this.nodeFilters[type] ? "visible" : "hidden");

        if (type === "unspecified" && !this.nodeFilters[type]) {
          d3.selectAll("line")
            .filter((d) => !d.target.type)
            .style("visibility", "hidden");
        }

        if (!this.nodeFilters[type]) {
          d3.selectAll("line")
            .filter((d) => d.source.type === type || d.target.type === type)
            .style("visibility", "hidden");
        } else {
          d3.selectAll("line").style("visibility", "visible");
        }
      } catch (error) {
        console.error("Error handling node filter change:", error);
      }
    },
    handleLinkFilterChange(type) {
      try {
        this.linkFilters[type] = !this.linkFilters[type];

        d3.selectAll("line")
          .filter(
            (d) =>
              d.source !== this.currentNode &&
              d.target !== this.currentNode &&
              d.type === type
          )
          .style("visibility", this.linkFilters[type] ? "visible" : "hidden");

        d3.selectAll("circle").style("visibility", (d) => {
          const isVisibleOutgoingNode =
            d3
              .selectAll("line")
              .filter((l) => l.source === d && this.linkFilters[l.type])
              .size() > 0;

          const isVisibleIncomingNode =
            d3
              .selectAll("line")
              .filter((l) => l.target === d && this.linkFilters[l.type])
              .size() > 0;

          return (this.nodeFilters[d.type] ||
            this.nodeFilters["unspecified"] ||
            !d.type) &&
            (isVisibleOutgoingNode || isVisibleIncomingNode)
            ? "visible"
            : "hidden";
        });
      } catch (error) {
        console.error("Error handling link filter change:", error);
      }
    },
  },
  created() {
    // Carica le opzioni del dropdown all'avvio
    this.loadOptions();
  },
};
</script>

<style>
@import "vue-multiselect/dist/vue-multiselect.css";

.graph-container {
  display: flex;
}

#my_dataviz_svg {
  flex: 1;
}

.tooltip {
  position: absolute;
  background-color: white;
  border: solid;
  border-width: 2px;
  border-radius: 5px;
  padding: 10px;
}

.legend-item {
  display: flex;
  align-items: center;
}

.submit-container {
  display: flex;
  justify-content: center;
  margin-top: 20px;
  margin-bottom: 20px;
}

.form-row-container {
  display: flex;
  align-items: center;
  margin-bottom: 10px;
}

.form-row-container:last-child {
  margin-bottom: 0;
}

.form-row-container .form-label {
  margin-right: 10px;
}
</style>

<template>
  <div>
    <div>
      <form @submit.prevent="search">
        <label>Select an id</label>
        <select id="dropdownId" multiple>
          <option v-for="node in originalNodes" :key="node.id" :value="node.id">{{ node.id }}</option>
        </select>
        <label>Select a country</label>
        <select id="dropdownCountry" multiple>
          <option v-for="country in uniqueCountries" :key="country" :value="country">{{ country }}</option>
        </select>
        <input type="submit" value="Submit">
      </form>
    </div>

    <!-- Grafico e Legenda -->
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
</template>


<script>
import * as d3 from 'd3';

const colorMapNodes = {
    "vessel": "#0000FF",
    "political_organization": "#FF0000",
    "person": "#800080",
    "organization": "#FFA500",
    "movement": "#FFFF00",
    "location": "#00FFFF",
    "event": "#008000",
    "company": "#A52A2A",
};
const colorMapLinks = {
    "ownership": "#FFD700",
    "partnership": "#98FB98",
    "family_relationship": "#FFA07A",
    "membership": "#87CEEB",
};

export default {
  data() {
    return {
      originalNodes: [], // Aggiungi un array per i nodi originali
      originalLinks: [], // Aggiungi un array per i link originali
      nodeFilters: { // Aggiungi un oggetto per i filtri dei nodi
        "vessel": true,
        "political_organization": true,
        "person": true,
        "organization": true,
        "movement": true,
        "location": true,
        "event": true,
        "company": true,
      }
    };
  },
  mounted() {
    //esportazione dei metodi
    this.loadOriginalData(); // Carica i dati originali al primo caricamento
    this.addSearchFunctionality();
    this.createLegend();
    // Chiamata a renderGraph senza argomenti
    this.renderGraph();
},
methods: {
    async loadOriginalData() {
        try {
            //caricamento del JSON
            const response = await fetch("/MC1.json");
            const mc1 = await response.json();

            // Salva i nodi e i link originali
            this.originalNodes = mc1.nodes;
            this.originalLinks = mc1.links;
        } catch (error) {
            console.error('Error loading original data:', error);
        }
    },
    async search() {
        try {
            //Recupera gli attributi selezionati 
            const selectedId = Array.from(document.getElementById('dropdownId').selectedOptions).map(option => option.value);
            const selectedCountry = Array.from(document.getElementById('dropdownCountry').selectedOptions).map(option => option.value);

            console.log(selectedId)
            console.log(selectedCountry)

            // Se non ci sono id o country selezionati, utilizza i nodi e i link originali
            if (selectedId.length === 0 && selectedCountry.length === 0) {
                this.renderGraph(this.originalNodes, this.originalLinks);
                return;
            }

            // Filtra i nodi da visualizzare in base agli id selezionati
            const filteredNodes = this.originalNodes.filter(node => selectedId.includes(node.id) || selectedCountry.includes(node.country));

            // Seleziona tutti i link collegati ai nodi filtrati
            const filteredLinks = this.originalLinks.filter(link => {
                // Verifica se il nodo sorgente e il nodo target sono entrambi inclusi nei nodi filtrati
                return filteredNodes.find(node => node.id === link.source) && filteredNodes.find(node => node.id === link.target);
            });
            console.log(filteredNodes)
            console.log(filteredLinks)
            // Pulisce il grafico
            d3.select("#my_dataviz_svg").selectAll("*").remove();

            // Renderizza il grafico con i nodi e i link filtrati
            this.renderGraph(filteredNodes, filteredLinks);
        } catch (error) {
            alert('Error performing search:', error);
        }
    },

    //metodo che costruisce il grafico 
    async renderGraph(nodes = this.originalNodes, links = this.originalLinks) {
        try {
            // Se nodes e links sono vuoti o undefined, carica i valori predefiniti da "/MC1.json"
            if (!nodes || !nodes.length || !links || !links.length) {
                const response = await fetch("/MC1.json");
                const mc1 = await response.json();
                nodes = mc1.nodes;
                links = mc1.links;
            }

            // Trova il nodo con il maggior numero di link
            const nodeLinksCount = {};
            links.forEach(link => {
                nodeLinksCount[link.source] = (nodeLinksCount[link.source] || 0) + 1;
                nodeLinksCount[link.target] = (nodeLinksCount[link.target] || 0) + 1;
            });

            const maxLinkCountNode = Object.keys(nodeLinksCount).reduce((a, b) => nodeLinksCount[a] > nodeLinksCount[b] ? a : b);

            // Seleziona tutti i link collegati al nodo con il maggior numero di link
            const filteredLinks = links.filter(link => link.source === maxLinkCountNode || link.target === maxLinkCountNode);

            // Seleziona tutti i nodi connessi ai link selezionati
            const connectedNodes = new Set();
            filteredLinks.forEach(link => {
                connectedNodes.add(link.source);
                connectedNodes.add(link.target);
            });

            // Inizializzazione del container SVG
            const width = 7000;
            const height = 7000;
            const svg = d3.create("svg")
                .attr("viewBox", [0, 0, width, height])
                .attr("width", "auto")
                .attr("height", "auto");

            // Inizializzazione dei nodi
            const node = svg.selectAll("circle")
                .data(nodes.filter(node => connectedNodes.has(node.id)))
                .enter().append("circle")
                .attr("r", 10)
                .attr("fill", d => colorMapNodes[d.type] || "grey") // Usa la mappa dei colori per i nodi
                .attr("cx", d => d.x)
                .attr("cy", d => d.y);

            // Inizializzazione dei link
            const link = svg.selectAll("line")
                .data(filteredLinks)
                .enter().append("line")
                .attr("stroke", d => colorMapLinks[d.type] || "grey") // Usa la mappa dei colori per i link
                .attr("x1", d => d.source.x)
                .attr("y1", d => d.source.y)
                .attr("x2", d => d.target.x)
                .attr("y2", d => d.target.y);

            // Inizializzazione del tooltip
            const Tooltip = d3.select("#my_dataviz_svg")
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
            const mouseover = function(event, d) {
                Tooltip
                    .style("opacity", 1)
                    .html(`
                        Name: ${d.id ? d.id : "N/A"}<br>
                        Country: ${d.country ? d.country : "N/A"}<br>
                        Type: ${d.type ? d.type : "N/A"}
                    `)
                    .style("left", (event.pageX) + "px")
                    .style("top", (event.pageY) + "px");
            };

            // Funzione che nasconde il tooltip se si ci si sposta col mouse da un nodo
            const mouseleave = function() {
                Tooltip
                    .transition()
                    .duration(200)
                    .style("opacity", 0);
            };

            // Funzione che gestisce lo zoom se si clicca su un nodo
            const clickNode = function(event, d) {
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
                const {x, y} = d;
                svg.transition().duration(750).call(
                    zoom.transform,
                    d3.zoomIdentity.translate(width / 2 - x, height / 2 - y).scale(64)
                );

                // Mostra il tooltip
                Tooltip
                    .style("opacity", 1)
                    .html(`
                        Name: ${d.id ? d.id : "N/A"}<br>
                        Country: ${d.country ? d.country : "N/A"}<br>
                        Type: ${d.type ? d.type : "N/A"}
                    `)
                    .style("left", (event.pageX) + "px")
                    .style("top", (event.pageY) + "px");

                // Esclude l'evento mouseleave
                Tooltip.on("mouseleave", null);
            };

            // Assegnazione delle funzioni ai nodi
            node.on("mouseover", mouseover)
                .on("mouseleave", mouseleave)
                .on("click", clickNode);

            // Inizializzazione delle forze e del comportamento dello zoom
            const simulation = d3.forceSimulation(nodes)
                .force("link", d3.forceLink(links).id(d => d.id))
                .force("charge", d3.forceManyBody().strength(-100))
                .force("center", d3.forceCenter(width / 2, height / 2));

            simulation.on("tick", () => {
                link
                    .attr("x1", d => d.source.x)
                    .attr("y1", d => d.source.y)
                    .attr("x2", d => d.target.x)
                    .attr("y2", d => d.target.y);

                node
                    .attr("cx", d => d.x)
                    .attr("cy", d => d.y);
            });

            const zoomed = (event) => {
                const {transform} = event;
                link.attr("transform", transform);
                node.attr("transform", transform);
            };

            //fuznione di szoom
            const zoom = d3.zoom()
                .scaleExtent([1, 128])
                .on("zoom", zoomed);

            svg.call(zoom);

            // Aggiungi il grafico al div
            document.getElementById("my_dataviz_svg").appendChild(svg.node());

        } catch (error) {
            console.error('Error rendering graph:', error);
        }
    },
    async loadOptions() {
        try {

          //recupera i nomi dei nodi 
            const response = await fetch("/MC1.json");
            const mc1 = await response.json();

            const dropdownId = document.getElementById('dropdownId');
            const dropdownCountry = document.getElementById('dropdownCountry');
            const nodes = mc1.nodes; 

            console.log('Number of nodes:', nodes.length); // Debug
            //carica i nodi nella select delle option
            if (nodes) {
                nodes.forEach(node => {
                    console.log('Country:', node.country); // Debug
                    const optionId = document.createElement('option');
                    optionId.value = node.id;
                    optionId.textContent = node.id;
                    dropdownId.appendChild(optionId);

                    // controllo di vuotezza del campo country
                    if (node.country && node.country.trim() !== '') {
                        const optionCountry = document.createElement('option');
                        optionCountry.value = node.country;
                        optionCountry.textContent = node.country;
                        dropdownCountry.appendChild(optionCountry);
                    }
                });
            }
        } catch (error) {
            console.error('Error loading options:', error);
        }
    },
    addSearchFunctionality() {
  try {
    const dropdownId = document.getElementById('dropdownId');
    const dropdownCountry = document.getElementById('dropdownCountry');
    const optionsId = dropdownId.getElementsByTagName('option');
    const optionsCountry = dropdownCountry.getElementsByTagName('option');
    const searchBarId = document.createElement('input');
    searchBarId.setAttribute('type', 'text');
    searchBarId.setAttribute('placeholder', 'Cerca per ID...');
    const searchBarCountry = document.createElement('input');
    searchBarCountry.setAttribute('type', 'text');
    searchBarCountry.setAttribute('placeholder', 'Cerca per country...');
    
    // Aggiungi la barra di ricerca per l'ID sopra al menu a discesa
    dropdownId.parentNode.insertBefore(searchBarId, dropdownId);
    
    // Filtra i valori unici per il campo "country"
    const uniqueCountries = Array.from(new Set(Array.from(optionsCountry).map(option => option.textContent)));
    
    // Aggiungi le opzioni filtrate per il campo "country" al menu a discesa
    uniqueCountries.forEach(country => {
      const option = document.createElement('option');
      option.value = country;
      option.textContent = country;
      dropdownCountry.appendChild(option);
    });
    
    // Aggiungi la barra di ricerca per il campo "country" sopra al menu a discesa
    dropdownCountry.parentNode.insertBefore(searchBarCountry, dropdownCountry);
    
    // Aggiungi un evento di ascolto per filtrare le opzioni del menu a discesa per l'ID
    searchBarId.addEventListener('input', () => {
      const searchText = searchBarId.value.toLowerCase();
      
      // Filtra le opzioni per l'ID
      for (let i = 0; i < optionsId.length; i++) {
        const optionText = optionsId[i].textContent.toLowerCase();
        if (optionText.indexOf(searchText) !== -1) {
          optionsId[i].style.display = '';
        } else {
          optionsId[i].style.display = 'none';
        }
      }
    });

    // Aggiungi un evento di ascolto per filtrare le opzioni del menu a discesa per il campo "country"
    searchBarCountry.addEventListener('input', () => {
      const searchText = searchBarCountry.value.toLowerCase();
      
      // Filtra le opzioni per il campo "country"
      for (let i = 0; i < optionsCountry.length; i++) {
        const optionText = optionsCountry[i].textContent.toLowerCase();
        if (optionText.indexOf(searchText) !== -1) {
          optionsCountry[i].style.display = '';
        } else {
          optionsCountry[i].style.display = 'none';
        }
      }
    });
  } catch (error) {
    console.error('Error adding search functionality:', error);
  }
},

    createLegend() {
        try {
            // Seleziona il div della legenda per i nodi
            const nodeLegendDiv = document.getElementById('my_node_legend');

            // Crea la legenda per i nodi
            Object.keys(colorMapNodes).forEach(type => {
                const nodeLegendItem = document.createElement('div');
                nodeLegendItem.style.display = 'flex';
                nodeLegendItem.style.alignItems = 'center';

                const nodeLegendColor = document.createElement('div');
                nodeLegendColor.style.width = '20px';
                nodeLegendColor.style.height = '20px';
                nodeLegendColor.style.backgroundColor = colorMapNodes[type];
                nodeLegendColor.style.marginRight = '5px';

                const nodeLegendLabel = document.createElement('label');
                nodeLegendLabel.textContent = type;

                const nodeLegendCheckbox = document.createElement('input');
                nodeLegendCheckbox.type = 'checkbox';
                nodeLegendCheckbox.checked = true;
                nodeLegendCheckbox.value = type;
                nodeLegendCheckbox.addEventListener('change', () => this.handleNodeFilterChange(type));

                nodeLegendItem.appendChild(nodeLegendColor);
                nodeLegendItem.appendChild(nodeLegendLabel);
                nodeLegendItem.appendChild(nodeLegendCheckbox);

                nodeLegendDiv.appendChild(nodeLegendItem);
            });

            // Seleziona il div della legenda per i link
            const linkLegendDiv = document.getElementById('my_link_legend');

            // Crea la legenda per i link
            Object.keys(colorMapLinks).forEach(type => {
                const linkLegendItem = document.createElement('div');
                linkLegendItem.style.display = 'flex';
                linkLegendItem.style.alignItems = 'center';

                const linkLegendColor = document.createElement('div');
                linkLegendColor.style.width = '20px';
                linkLegendColor.style.height = '20px';
                linkLegendColor.style.backgroundColor = colorMapLinks[type];
                linkLegendColor.style.marginRight = '5px';

                const linkLegendLabel = document.createElement('label');
                linkLegendLabel.textContent = type;

                linkLegendItem.appendChild(linkLegendColor);
                linkLegendItem.appendChild(linkLegendLabel);

                linkLegendDiv.appendChild(linkLegendItem);
            });
        } catch (error) {
            console.error('Error creating legend:', error);
        }
    },
    handleNodeFilterChange(type) {
    try {
        // Aggiorna lo stato del filtro del nodo o del link
        this.nodeFilters[type] = !this.nodeFilters[type];

        // Rendi invisibili i nodi che non corrispondono ai filtri
        d3.selectAll("circle")
            .filter(d => d.type === type)
            .style("visibility", this.nodeFilters[type] ? "visible" : "hidden");

        // Rendi invisibili i link che non corrispondono ai filtri
        d3.selectAll("line")
            .filter(d => d.type === type)
            .style("visibility", this.nodeFilters[type] ? "visible" : "hidden");
    } catch (error) {
        console.error('Error handling node filter change:', error);
    }
  },
  },
  computed: {
    uniqueCountries() {
      return Array.from(new Set(this.originalNodes.map(node => node.country)));
    },
  },
  created() {
    // Carica le opzioni del dropdown all'avvio
    this.loadOptions();
  },
};
</script>

<style>
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
</style>

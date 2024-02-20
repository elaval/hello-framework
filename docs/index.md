---
theme: dashboard
title: PIB per capita
toc: false
---


<style>

.hero {
  display: flex;
  flex-direction: column;
  align-items: center;
  font-family: var(--sans-serif);
  margin: 4rem 0 8rem;
  text-wrap: balance;
  text-align: center;
}

.hero h1 {
  margin: 2rem 0;
  max-width: none;
  font-size: 14vw;
  font-weight: 900;
  line-height: 1;
  background: linear-gradient(30deg, var(--theme-foreground-focus), currentColor);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.hero h2 {
  margin: 0;
  max-width: 34em;
  font-size: 20px;
  font-style: initial;
  font-weight: 500;
  line-height: 1.5;
  color: var(--theme-foreground-muted);
}

@media (min-width: 640px) {
  .hero h1 {
    font-size: 90px;
  }
}

</style>

<div class="hero">
  <h1>PIB per cápita</h1>  
  <p>Comparación de PIB per cápita de Chile con países de similar ingreso en ${targetYear}</p>
</div>


```js
import {es_ES} from "./components/config.js";
d3.formatDefaultLocale(es_ES);
```

```js
const data = FileAttachment("./data/dataPIBPerCapita.tsv").tsv();
```

```js
const targetYear = view(Inputs.range([2000, 2022], {
  label: "Año de referencia",
  step: 1,
  value: 2013
}));

const comparisonYear = 2022;
```


```js
const umbralDiferencia = view(Inputs.range([0,30], {
  label: "Umbral de diferencia (%) para considerar paises como similares",
  step: 1,
  value: 10
}));
```


```js
const windowDiff = umbralDiferencia / 100;

const referenceIncome = data.find(
  (d) => d["countryCode"] == "CHL" && d.year == targetYear
)["value"];

const dataPlot = data.filter((d) => d.year == targetYear);

const similarCountries = data
  .filter(
    (d) =>
      d.year == targetYear &&
      d.value >= referenceIncome * (1 - windowDiff) &&
      d.value <= referenceIncome * (1 + windowDiff)
  )
  .map((d) => d["countryCode"]);

const dataSimilar = data.filter((d) => similarCountries.includes(d.countryCode));

const dataSimilarPorPais = _.chain(dataSimilar)
.groupBy(d => d.country)
.map((items,key) => {
  return {
    country:key,
    targetValue: items.find(d => d.year == targetYear)["value"],
    comparisonValue: items.find(d => d.year == comparisonYear) && items.find(d => d.year == comparisonYear)["value"] || null,    
  }
}).value();

  dataSimilarPorPais.forEach(d => {
    d.growth = d.comparisonValue && d.targetValue ? d.comparisonValue/d.targetValue - 1 : null
  })

```

```js
const plot1 = Plot.plot({
  title: `Evolucion de PIB per cápita entre 2000 y 2022`,

  width:width,
  marginLeft: 50,
  marginRight: 100,
  y: { type: "log" ,label:'PIB per cápita, PPA (2017 const.)'},
  x: { type: "linear" ,label:'Año'},

  marks: [
    Plot.ruleY([0]),
    Plot.ruleX([targetYear]),
    Plot.lineY(dataSimilar, {
      x: d => +d["year"],
      y: d => +d["value"],
      z: "country",
      stroke: (d) => (d.countryCode == "CHL" ? "blue" : "grey"),
      strokeWidth: d => (d.countryCode == "CHL" ? 2 : 1),
      sort: d => (d.countryCode == "CHL" ? 0 : 1)
    }),
    Plot.text(
      dataSimilar,
      Plot.selectLast({
        x: "year",
        y: "value",
        z: "country",
        text: "country",
        textAnchor: "start",
        fill: (d) => (d.countryCode == "CHL" ? "blue" : "grey"),
        dx: 5,
              sort: d => (d.countryCode == "CHL" ? 0 : 1),
              fontWeight: d => (d.countryCode == "CHL" ? "bold" : "bold")

      })
    )
  ]
})
```

```js

  const dataPlotTarget = dataSimilar.filter((d) => d.year == targetYear);
  const dataPlot2022 = dataSimilar.filter((d) => d.year == 2022);

```


```js
const pTarget = Plot.plot({
    title: `PIB per cápita ${targetYear}`,

    marginLeft: 150,
    marks: [
      Plot.barX(dataPlotTarget, {
        x: "value",
        y: "country",
        sort: { y: "x", reverse: true },
        fill: (d) => (d.countryCode == "CHL" ? "blue" : "grey")
      }),
      Plot.ruleX([0])
    ]
})
```

```js
const p2022 = Plot.plot({
  title: `PIB per cápita en 2022`,
    marginLeft: 150,
    width:width,
    color:{legend:true, range:["orange"]},
    x:{label:'PIB per cápita, PPA (2017 const.)', grid:true},
    y:{label:'País'},
    marks: [
      Plot.barX(dataPlot2022, {
        x: "value",
        y: "country",
        sort: { y: "x", reverse: true },
        fill: (d) => (d.countryCode == "CHL" ? "blue" : "grey")
      }),
      Plot.tickX(dataPlotTarget, {
        x: d => +d["value"],
        y: "country",
        sort: { y: "x", reverse: true },
        stroke: d => `Referencia ${targetYear}`,
        strokeWidth:3
      }),
      Plot.ruleX([0])
    ]
})
```



<div class="card grid-colspan-2">
<div>En ${targetYear} el PIB per capita de Chile era ${d3.format("$,d")(referenceIncome)} medido en términos de Paridad de Poder Adquisitivo (PPA) a precios constantes de 2017 en dólares internacionales.<p>Los países con un ingreso similar de +/- ${d3.format(".0%")(windowDiff)} (${d3.format("$,d")(referenceIncome * (1 - windowDiff))} a ${d3.format("$,d")(referenceIncome * (1 + windowDiff))}) eran:</div>
${html`
<table>
<tr><th>País</th><th>PIB per capita ${targetYear}</th><th>PIB per capita ${comparisonYear}</th><th>Crecimiento (%)</th></tr>
${_.chain(dataSimilarPorPais).sortBy(d => d.growth).map((d) => html`<tr><td>${d.country}</td><td>${d3.format("$,d")(d.targetValue)}</td><td>${d.comparisonValue && d3.format("$,d")(d.comparisonValue) || ""}</td><td>${d.growth && d3.format(".2%")(d.growth) || ""}</td></tr>`).value()}
</table>`}<p>PIB per cápita, PPA (2017 const.)</p></div>


<div class="card">${plot1}</div>
<div class="card">${p2022}</div>


**Indicador**
PIB per capita, PPP (constant 2017 international $)

GDP per capita based on purchasing power parity (PPP). PPP GDP is gross domestic product converted to international dollars using purchasing power parity rates. An international dollar has the same purchasing power over GDP as the U.S. dollar has in the United States. GDP at purchaser's prices is the sum of gross value added by all resident producers in the country plus any product taxes and minus any subsidies not included in the value of the products. It is calculated without making deductions for depreciation of fabricated assets or for depletion and degradation of natural resources. Data are in constant 2017 international dollars.

**Fuente** International Comparison Program, World Bank | World Development Indicators database, World Bank | Eurostat-OECD PPP Programme. https://databank.worldbank.org/source/world-development-indicators/Series/NY.GDP.PCAP.PP.KD


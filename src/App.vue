<script setup>
import { geoMercator, geoPath } from "d3-geo";
import { select } from "d3-selection";
import { feature, mesh } from "topojson-client";
import { onMounted } from "vue";
import allDistricts from "./assets/districts.json";
import allCities from "./assets/cities.json";

const countryLabelOffset = [-0.2, 0];

const districts = {
  ...allDistricts,
  objects: {
    "afghanistan-districts": allDistricts.objects["afghanistan-districts"],
  },
};

const visibleCities = [
  { ADM0NAME: "Afghanistan", NAME: "Herat" },
  { ADM0NAME: "Afghanistan", NAME: "Kandahar" },
  { ADM0NAME: "Afghanistan", NAME: "Kabul" },
  { ADM0NAME: "Afghanistan", NAME: "Gardiz" },
  { ADM0NAME: "Afghanistan", NAME: "Baghlan" },
  { ADM0NAME: "Afghanistan", NAME: "Kondoz" },
  { ADM0NAME: "Afghanistan", NAME: "Jalalabad" },
];

const cities = {
  ...allCities,
  features: allCities.features.filter((city) => {
    return (
      visibleCities.filter((vizCity) => {
        return (
          city.properties.ADM0NAME === vizCity.ADM0NAME &&
          city.properties.NAME === vizCity.NAME
        );
      }).length > 0
    );
  }),
};

const talibanDistricts = [
  "Aliabad",
  "Azra",
  "Baghran",
  "Baharak",
  "Bala Buluk",
  "Bangi",
  "Baraki Barak",
  "Chahar Dara",
  "Charkh",
  "Charsada",
  "Dara",
  "Dashte Archi",
  "Dishu",
  "Ghorak",
  "Ishkamish",
  "Jawand",
  "Kakar",
  "Khaki Safed",
  "Kham Ab",
  "Khanabad",
  "Kharwar",
  "Kohistanat",
  "Nawa",
  "Pashtun Kot",
  "Registan",
  "Saydabad",
  "Shorabak",
  "Tagab",
  "Warduj\n",
  "Waygal",
  "Waza Khwa",
  "Yamgan (Girwan)",
  "Yangi Qala",
  "Zurmat",
];

const contestedDistricts = [
  "Ajristan",
  "Argo",
  "Baghlani Jadid",
  "Barmal",
  "Dahana-I- Ghuri",
  "Dangam",
  "Darayim",
  "Dihrawud",
  "Dila",
  "Ghazni",
  "Gizab",
  "Gulistan",
  "Gurziwan",
  "Imam Sahib",
  "Jurm",
  "Kajaki",
  "Khas Uruzgan",
  "Khwaja Ghar",
  "Kishim",
  "Kunduz",
  "Marawara",
  "Musa Qala",
  "Naw Zad",
  "Nijrab",
  "Nika",
  "Qalay-I- Zal",
  "Qaysar",
  "Sangin",
  "Shahidi Hassas",
  "Shindand",
  "Sozma Qala",
  "Tala Wa Barfak",
  "Tulak",
  "Urgun",
  "Yahya Khel",
  "Yosuf Khel",
  "Ziruk",
];

const mapId = `map-${crypto.randomUUID()}`;

const width = 1000;
const height = 1000;

const projection = geoMercator();
const path = geoPath(projection);

const districtFeatures = feature(
  districts,
  districts.objects["afghanistan-districts"]
).features;

// Get bounds for raster scaling
const bounds = path.bounds(
  feature(districts, districts.objects["afghanistan-districts"])
);

// Set scale for raster image
const scale =
  1 /
  Math.max(
    (bounds[1][0] - bounds[0][0]) / width,
    (bounds[1][1] - bounds[0][1]) / height
  );

// Set dimensions and translation for raster image
const rWidth = (bounds[1][0] - bounds[0][0]) * scale;
const rHeight = (bounds[1][1] - bounds[0][1]) * scale;
const rTranslateX = (width - rWidth) / 2;
const rTranslateY = (height - rHeight) / 2;

// Country outline
const country = mesh(
  districts,
  districts.objects["afghanistan-districts"],
  function (a, b) {
    return a === b;
  }
);

// Fit projection to country
projection.fitSize([width, height], country);

onMounted(() => {
  const svg = select(`#${mapId}`);

  // Relief map (raster)
  svg
    .append("image")
    .attr("xlink:href", "afghanistan.png")
    .attr("width", rWidth)
    .attr("height", rHeight)
    .attr("transform", `translate(${rTranslateX} ${rTranslateY})`);

  // Districts
  svg
    .selectAll(".district")
    .data(districtFeatures)
    .join("path")
    .attr("class", "district")
    .attr("opacity", 0.8)
    .attr("stroke", "#6E6E6E")
    .attr("stroke-width", 0.5)
    .attr("stroke-opacity", 0.2)
    .attr("fill", (d) => {
      if (talibanDistricts.indexOf(d.properties.DIST_34_NA) > -1)
        return "#CC0000";

      if (contestedDistricts.indexOf(d.properties.DIST_34_NA) > -1)
        return "#FDBF4F";

      return "white";
    })
    .attr("d", path);

  // City markers groups
  const cityMarker = svg
    .selectAll(".city")
    .data(cities.features)
    .join("g")
    .attr("class", "city")
    .attr(
      "transform",
      (d) => `translate(${projection(d.geometry.coordinates)})`
    );

  // City marker circles
  cityMarker
    .append("circle")
    .attr("r", 4)
    .attr("fill", "none")
    .attr("stroke", "#000000")
    .attr("opacity", 0.6)
    .attr("stroke-width", 2);

  // City marker labels
  cityMarker
    .append("text")
    .text((d) => d.properties.NAME)
    .attr("transform", "translate(6, 12)")
    .attr("font-size", 18)
    .attr("alignment-baseline", "middle")
    .attr("font-family", "Helvetica, Arial, sans-serif")
    .attr("opacity", 0.6);

  // Country outline
  svg
    .append("path")
    .attr("class", "country")
    .datum(country)
    .attr("fill", "none")
    .attr("stroke", "#6E6E6E")
    .attr("stroke-opacity", 0.7)
    .attr("d", path);

  // Country label
  svg
    .append("text")
    .text(cities.features[0].properties.ADM0NAME.toUpperCase())
    .attr(
      "transform",
      `translate(
        ${width / 2 + (width / 2) * countryLabelOffset[0]}
        ${height / 2 + (height / 2) * countryLabelOffset[1]}
      )`
    )
    .attr("font-family", "Helvetica, Arial, sans-serif")
    .attr("font-weight", 500)
    .attr("text-anchor", "middle")
    .attr("alignment-baseline", "middle")
    .attr("opacity", 0.4)
    .attr("font-size", 32)
    .attr("letter-spacing", 6);
});
</script>

<template>
  <svg :id="mapId" :viewBox="`0 0 ${width} ${height}`"></svg>
</template>

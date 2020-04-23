# Command-line Cartography [UK edition]

This repo aims to run through [Mike Bostock's](https://bost.ocks.org/mike/) Command-line Cartography series - creating a population density choropleth on the command-line using javascript packages. The original series mapped sunny California. We'll map the soggy UK instead. Most of the wit and wisdom has been suppressed (and Mike's a bit of a dude) so be sure to check out the original:
+ [Part 1](https://medium.com/@mbostock/command-line-cartography-part-1-897aa8f8ca2c)
+ [Part 2](https://medium.com/@mbostock/command-line-cartography-part-2-c3a82c5c0f3)
+ [Part 3](https://medium.com/@mbostock/command-line-cartography-part-3-1158e4c55a1e)
+ [Part 4](https://medium.com/@mbostock/command-line-cartography-part-4-82d0d26df0cf)

![Teaser image](./images/uk.svg)

## Prerequisites
I'm running this through Windows PowerShell. You'll need npm and node, and some basic familiarity with command-line. 
The instructions below constitute a few minor modifications and hacks that I needed to make in order to get everything to work, as well as sources for
the UK geography and shapefiles.

IMPORTANT: if (like me) you're working through on Powershell, for each ```command``` below, you'll need to write in the command as:

```cmd /c 'command'```

## Part 1: Get geography and project

```
shp2json Lower_Layer_Super_Output_Areas_December_2011_Boundaries_EW_BSC.shp -o uk.json
geoproject "d3.geoMercator().fitSize([960, 960], d)" < uk.json > uk-merc.json
geoproject "d3.geoIdentity().fitSize([960, 960], d)" < uk.json > uk-ident.json #alternative 
geo2svg -w 960 -h 960 < uk-merc.json > uk-merc.svg
```

## Part 2: Get data and join

```
ndjson-split "d.features" < uk-merc.json > uk-merc.ndjson
ndjson-map "d.id = d.properties.LSOA11CD, d" < uk-merc.ndjson > uk-merc-id.ndjson
csv2json -n lsoa-data.csv > lsoa-data.json
ndjson-map "{id:d.LSOA, dens:d[""Pop_dens_hect""]}" < lsoa-data.json > lsoa-data-dens.ndjson
ndjson-join "d.id" uk-merc-id.ndjson lsoa-data-dens.ndjson > uk-merc-join.ndjson
ndjson-map "d[0].properties = {density: d[1].dens}, d[0]" < uk-merc-join.ndjson > uk-merc-density.ndjson
ndjson-reduce < uk-merc-density.ndjson | ndjson-map "{type:""FeatureCollection"", features:d}" > uk-merc-density.json
ndjson-map -r d3 "(d.properties.fill = d3.scaleSequential(d3.interpolateViridis).domain([0, 50])(d.properties.density), d)" < uk-merc-density.ndjson > uk-merc-colour.ndjson
geo2svg -n --stroke=none -p 1 -w 960 -h 960 < uk-merc-colour.ndjson > uk-merc-colour.svg
```

## Part 3: Topology for the browser

```
geo2topo -n tracts=uk-merc-density.ndjson > uk-tracts-topo.json
toposimplify -p 1 -f < uk-tracts-topo.json > uk-simple-topo.json
topoquantize 1e5 < uk-simple-topo.json > uk-quantized-topo.json
topomerge lauth=tracts -k 'd.properties.la' < uk-quantized-topo.json > uk-authmerge-topo.json
topomerge --mesh -f 'a !== b' lauth=lauth < uk-authmerge-topo.json > uk-topo.json
```

## Part 4: Expressive scales

```
topo2geo lauth=- < uk-topo.json | ndjson-map -r d3 "z = d3.scaleSequential(d3.interpolateViridis).domain([0, 8000]), d.features.forEach(f=>f.properties.fill = z(f.properties.density)), d" | ndjson-split "d.features" | geo2svg -n --stroke none -p 1 -w 960 -h 960 > uk-lauth-color.svg
topo2geo tracts=- < uk-topo.json | ndjson-map -r d3 "z = d3.scaleSequential(d3.interpolateViridis).domain([0, 10]), d.features.forEach(f=>f.properties.fill = z(Math.sqrt(f.properties.density))), d" | ndjson-split "d.features" | geo2svg -n --stroke none -p 1 -w 960 -h 960 > uk-tracts-sqrt.svg
topo2geo tracts=- < uk-topo.json | ndjson-map -r d3 -r d3-scale-chromatic "z = d3.scaleThreshold().domain([1, 2, 5, 10, 20, 50]).range(d3.schemeOrRd[9]), d.features.forEach(f=>f.properties.fill=z(f.properties.density)), d" | ndjson-split "d.features" | geo2svg -n --stroke none -p 1 -w 960 -h 960 > uk-tracts-threshold.svg
```

```
topo2geo tracts=- < uk-topo.json | ndjson-map -r d3 -r d3=d3-scale-chromatic "z = d3.scaleThreshold().domain([1, 2, 5, 10, 20, 50, 10]).range(d3.schemeOrRd[9]), d.features.forEach(f => f.properties.fill = z(f.properties.density)), d" | ndjson-split "d.features" > uk-geo-init.json
topo2geo lauth=- < uk-topo.json | ndjson-map "d.properties = {""stroke"": ""#000"", ""stroke-opacity"": 0.3}, d" >> uk-geo-init.json
geo2svg -n --stroke none -p 1 -w 960 -h 960 < uk-geo-init.json > uk.svg
```

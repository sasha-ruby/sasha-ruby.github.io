---
layout: post
title: English Premier League 2014-2015 - Chelsea's lead by round
thumbnailurl: /images/epl_2014_15_leader_pointdiff250.png
category: news
tags: [dataviz, sportstats, d3js]
published: true
shortUrl: 
---

This chart was originally published on [SportStats dot Click](http://sportstats.click).

This chart highlights how comfortable Chelsea FC was in its run to English Premier League title in season 2014-2015. A couple of facts:

- Chelsea was in lead from round 1, and the only time it wasn’t in top position was round 2, when Tottenham took the lead based on goal difference only.
- The only time some other team got really close is round 20, when Manchester City had the same number of points (Chelsea lost 5:3 to Tottenham). The ascent began in the very next round, and after only 4 matches, Chelsea was 7 points ahead of City.

> You can hover over chart points to highlight difference at that point.
    
<style>
rect.dimple-tooltip {
    font-size: 12px;
    line-height: 18px;
}
</style>

<div id="epl_2014_15_leader_pointdiff"></div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/d3/3.5.5/d3.min.js"></script>
<script src="http://dimplejs.org/dist/dimple.v2.1.2.min.js"></script>
<script type="text/javascript">
var svg = dimple.newSvg("#epl_2014_15_leader_pointdiff", 600, 450);
d3.csv("/data/epl_2014_2015_leader_pointdiff.csv", function (data) {

    // Create and Position a Chart
    var myChart = new dimple.chart(svg, data);
    myChart.setBounds(60, 20, 530, 390);
    var x = myChart.addCategoryAxis("x", "Matches Played")
    myChart.addMeasureAxis("y", "Points Difference");

    x.fontSize = "9";
    x.title = "Round"

    // Order the x axis by date
    x.addOrderRule("Matches Played");

    // Points diff color coded
    myChart.addColorAxis("Points Difference", ["red", "gold", "green", "darkgreen"]);

    // Add a thick line with markers
    var lines = myChart.addSeries(null, dimple.plot.line);

    lines.lineWeight = 4;
    lines.lineMarkers = true;

    // Draw the chart
    myChart.draw();

});
</script>

---
layout: post
title: English Premier League 2014-2015 Roundchart
thumbnailurl: /images/epl-2014-2015-roundchart250.png
category: news
tags: [dataviz, sportstats, d3js]
published: true
shortUrl: http://bit.ly/1AYAT4c
---

This chart was originally published on [SportStats dot Click](http://sportstats.click).

Chelsea won its 4th title in season 2014/2015. This was 23rd season since English Premier League was established. While still far away from accomplishments of Manchester United (13 titles), the way Chelsea was dominating the league this season was really impressive. They have been the leader from round 1, and round 2 was the only time they let some other team lead the pack (Tottenham Hotspur).

The following interactive chart shows teams positions round by round.

> You can hover over team names to highlight that specific team and how it was placed throughout the season.  You can also zoom in/out specific round by simply clicking that round column.
    
<style>
text {
    font-family: Verdana, sans-serif;
}
text.lap {
    fill: #baa;
    font-size:7px;
}

text.label {
    fill: #000000;
    font-weight: normal;
    font-size: 9px;
}

text.label.marker {
    font-weight: bold;
    font-size:10px;
}

polyline.placing {
    fill: none;
    stroke-width: 4;
}

line.tickLine {
    stroke: #c9c9c9;
}

.lapped {
    background-color: #333333;
    stroke: #333333;
    fill: #333333;
}

.safety{
    background-color: #330000;
    stroke: #330000;
    fill: #330000;
}
</style>
<script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/d3/3.5.5/d3.min.js"></script>

<script type="text/javascript">
// Dimensions.
var DIMENSIONS = getWindowDimensions();
//var WIDTH = DIMENSIONS.width - 100;    // 20 => padding.
//var HEIGHT = DIMENSIONS.height - 100;  // 50 => legend, title and padding.
var WIDTH = 756;    // 20 => padding.
var HEIGHT = 600;  // 50 => legend, title and padding.

// Insets.
var INSETS = {'left': 170, 'right': 170, 'top': 50, 'bottom': 50};

// Padding.
var PADDING = {'left': 20, 'right': 20, 'top': 15, 'bottom': 15};

// Tick-mark length.
var TICK_MARK_LENGTH = 8;

// Marker radius.
var MARKER_RADIUS = 10;

// Scales.
var SCALES = {};

// Transition duration.
var TRANSITION_DURATION = 1000;

// Opacity of dimmed and highlighted objects.
var REGULAR_OPACITY = 0.75;
var DIMMED_OPACITY = 0.05;
var HIGHLIGHT_OPACITY = 1.0;
var DIMMED_OPACITY_TEXT = 0.25;
var HIGHLIGHT_OPACITY_TEXT = 1.0;

// Zoom factors.
var ZOOM_PEAK = 6.0;
var ZOOM_SHOULDER = 3.0;

// Zooming.
var zoomed = false;

// Visualize when document has loaded.
//
window.onload = function() {

    // Load data.
    d3.json("/data/epl_2014_2015_roundchart.json", function(data) {

        // Check integrity.
        integrityCheck(data);

        // Sort laps on finishing order.
        data.laps.sort(function(a, b) {

            var aLaps = a.placing.length;
            var bLaps = b.placing.length;
            return aLaps == bLaps ? a.placing[aLaps - 1] - b.placing[bLaps - 1] : bLaps - aLaps;
        });

        // Process lap markers..
        data.pitstops = processLapMarkers(data, "pitstops");
        data.mechanical = processLapMarkers(data, "mechanical");
        data.accident = processLapMarkers(data, "accident");

        // Visualize the data.
        visualize(data);
    });
};

// Check data.
//
// data: the data to check.
//
function integrityCheck(data) {
    var laps = data.laps;
    var lapCount = data.lapCount;

    // Check lap data.
    checkLaps(laps, lapCount);
}
// Check lap data.
//
// laps: the lap data.
// lapCount: number of laps.
//
function checkLaps(laps, lapCount) {

    for (var j = 0;
         j < laps.length;
         j++) {

        // Has name?
        var name = laps[j].name;
        if (name == undefined || name.length == 0) {

            alert("Warning: invalid name for element " + j);
        }
        // Has color?
        var color = laps[j].color;
        if (color == undefined || color.length == 0) {

            alert("Warning: invalid color for element " + j);
        }
		
        // Has placings?
        var places = laps[j].placing;
        if (places == undefined) {

            alert("Warning: missing placings for element " + j + " (" + name + ")");
        }
        else if (places.length == 0 || places.length > lapCount + 1) {

            alert("Warning: invalid number of placings (" + places.length + ") for element " + j +
                " (" + name + ") - expected between 1 and " + (lapCount - 1));
        }

        // Check markers.
        var maxLaps = places.length;
        checkMarker(laps[j].pitstops, "pitstop", maxLaps, j, name);
        checkMarker(laps[j].mechanical, "mechanical", maxLaps, j, name);
        checkMarker(laps[j].accident, "accident", maxLaps, j, name);
    }

    for (var i = 0;
         i < lapCount;
         i++) {

        var positions = [];
        for (j = 0;
             j < laps.length;
             j++) {

            places = laps[j].placing;
            if (places.length > i) {

                // Valid placing?
                var placing = places[i];
                if (isNaN(placing) || placing < 1 || placing % 1 != 0) {

                    alert("Warning: invalid placing '" + placing + "' for " + laps[j].name)
                }
                else {

                    var count = positions[placing];
                    positions[placing] = isNaN(count) ? 1 : count + 1
                }
            }
        }

        // Check for duplicate/missing positions.
        for (j = 1;
             j < positions.length;
             j++) {

            count = positions[j];
            if (count != 1) {

                alert("Warning: data inconsistent: lap " + i + ", position " + j + ", count " + count);
            }
        }
    }
}

// Check integrity of marker data.
//
// marker: marker data.
// name: driver name.
// type: text description of marker.
// max: maximum allowed lap value of marker.
// index: index of driver in list.
//
function checkMarker(marker, type, max, index, name) {

    if (marker != undefined) {

        // Check marker.
        for (var i = 0;
             i < marker.length;
             i++) {

            var stop = marker[i];
            if (isNaN(stop) || stop < 0 || stop >= max || stop % 1 != 0) {

                alert("Warning: invalid " + type + " (" + stop + ") for element " + index + " (" + name + ")");
            }
        }
    }
}

// Check lapped data.
//
// lapped: the lapped data.
// lapCount: number of laps.
// driverCount: number of drivers.
//
function checkLapped(lapped, lapCount, driverCount) {

    if (lapped != undefined) {

        var lappedLength = lapped.length;
        if (lappedLength != lapCount) {

            alert("Lapped array length (" + lappedLength + ") incorrect - expected length " + lapCount);
        }

        for (var j = 1;
             j < lappedLength;
             j++) {

            // Valid position.
            var position = lapped[j];
            if (isNaN(position) || position % 1 != 0 || position < -1 || position > driverCount) {

                alert("Invalid lapped position: element " + j + " (" + position
                    + "); expected integer between -1 and " + driverCount);
            }
        }
    }
}

// Check safety car data.
//
// safety: safety car data.
// lapCount: number of laps.
//
function checkSafetyCar(safety, lapCount) {

    if (safety != undefined) {

        for (var i = 0;
             i < safety.length;
             i++) {

            // Valid lap?
            var lap = safety[i];
            if (isNaN(lap) || lap < 0 || lap % 1 != 0 || lap > lapCount) {

                alert("Invalid safety car lap: element " + i + " (" + lap
                    + "); expected integer between 0 and " + lapCount);
            }
        }
    }
}

// Process lap markers.
//
// data: lap data.
// key: marker key.
//
function processLapMarkers(data, key) {

    var markers = [];
    var p = 0;
    for (var i = 0;
         i < data.laps.length;
         i++) {

        var lapData = data.laps[i];
        var laps = lapData[key];
        if (laps != undefined) {
            for (var j = 0;
                 j < laps.length;
                 j++) {

                var lap = laps[j];
                var marker = {};
                marker.start = lapData.placing[0];
                marker.lap = lap;
                marker.placing = lapData.placing[lap];
                marker.name = lapData.name;

                markers[p++] = marker;
            }
        }
    }
    return markers;
}

// Create the visualization.
//
// data the lap data object.
//
function visualize(data) {

    // Configure scales.
    configureScales(data);

    var vis = d3.select('#chart')
        .append('svg:svg')
        .attr('width', WIDTH)
        .attr('height', HEIGHT)
        .attr('class', 'zoom');

    // Background rect to catch zoom clicks.
    vis.append('svg:rect')
        .attr('class', 'zoom')
        .attr('x', 0)
        .attr('y', 0)
        .attr('width', WIDTH)
        .attr('height', HEIGHT)
        .style('opacity', 0.0);

    // Add safety car element.
    addSafetyElement(vis, data.safety);

    // Add lapped element.
    addLappedElement(vis, data.lapped);

    // Lap tick-lines.
    addLapTickLines(vis, data.lapCount);

    // Lap labels.
    addLapLabels(vis, data.lapCount, SCALES.y.range()[0] - PADDING.bottom, '0.15em', 'top');
    addLapLabels(vis, data.lapCount, SCALES.y.range()[1] + PADDING.top, '0.15em', 'bottom');

    // Add placings poly-lines.
    addPlacingsLines(vis, data.laps);

    // Add name labels.
    addDriverLabels(vis, data.laps, 'pole', SCALES.x(0) - PADDING.right, 'end')
        .attr('y', function (d) {

            return SCALES.y(d.placing[0] - 1);
        });
    addDriverLabels(vis, data.laps, 'flag', SCALES.x(data.lapCount) + PADDING.left+5, 'start')
        .attr('y', function (d, i) {

            return SCALES.y(i);
        });

    // Add markers.
    addMarkers(vis, data.pitstops, "pitstop", "P");
    addMarkers(vis, data.mechanical, "mechanical", "M");
    addMarkers(vis, data.accident, "accident", "X");

    // Listen for clicks -> zoom.
    vis.selectAll('.zoom')
        .on("click", function() {

            toggleZoom(vis, d3.mouse(this)[0]);
        });
}

// Configure the scales.
//
// data: data set.
//
function configureScales(data) {

    SCALES.x = d3.scale.linear()
        .domain([0, data.lapCount])
        .range([INSETS.left, WIDTH - INSETS.right]);

    SCALES.y = d3.scale.linear()
        .domain([0, data.laps.length - 1])
        .range([INSETS.top, HEIGHT - INSETS.bottom]);

    SCALES.clr = d3.scale.category20();
}

// Highlight driver.
//
// vis: the data visualization root.
// index: index of driver to highlight.
//
function highlight(vis, name) {

    // Dim others.
    vis.selectAll('polyline').transition()
        .duration(500)
        .delay(100)
//        .classed("highlighted", function(d){
//            return d.name == name ? HIGHLIGHT_OPACITY : DIMMED_OPACITY;
//        })
//            classed("highglighted", true)
//            classed("highglighted", false)
        .style('opacity', function(d) {
            return d.name == name ? HIGHLIGHT_OPACITY : DIMMED_OPACITY;
        });

    vis.selectAll('circle').transition()
        .duration(500)
        .delay(100)
        .style('opacity', function(d) {
            return d.name == name ? HIGHLIGHT_OPACITY : DIMMED_OPACITY;
        });

    vis.selectAll('text.label').transition()
        .duration(500)
        .delay(100)
        .style('opacity', function(d) {
            return d.name == name ? HIGHLIGHT_OPACITY_TEXT : DIMMED_OPACITY_TEXT;
        });
}

// Remove highlights.
//
// vis: the data visualization root.
//
function unhighlight(vis) {

    // Reset opacity.
    vis.selectAll('polyline').transition()
        .duration(500)
        .delay(100)
        .style('opacity', REGULAR_OPACITY);
    vis.selectAll('circle').transition()
        .duration(500)
        .delay(100)
        .style('opacity', HIGHLIGHT_OPACITY);
    vis.selectAll('text.label').transition()
        .duration(500)
        .delay(100)
        .style('opacity', HIGHLIGHT_OPACITY);
}

// Zoom/unzoom.
//
// vis: the data visualization root.
// mouseX: x-coordinate of mouse click.
//
function toggleZoom(vis, mouseX) {

    // Get lap of mouse-click position.
    var lap = Math.round(SCALES.x.invert(mouseX));

    // Clamp to domain.
    var domain = SCALES.x.domain();
    lap = Math.max(domain[0], Math.min(domain[1], lap));

    // Specify transform.
    var xform = zoomed ? unzoomXform : zoomXform;
    zoomed = !zoomed;

    // Transition tick lines.
    vis.selectAll('line.tickLine')
        .transition()
        .duration(TRANSITION_DURATION)
        .attr("x1", function(d) {

            return SCALES.x(xform(d + 0.5, lap))
        })
        .attr("x2", function(d) {

            return SCALES.x(xform(d + 0.5, lap))
        });

    // Transition tick labels.
    vis.selectAll('text.lap')
        .transition()
        .duration(TRANSITION_DURATION)
        .attr("x", function(d) {

            return SCALES.x(xform(d, lap))
        });

    // Transition safety elements.
    vis.selectAll('rect.safety')
        .transition()
        .duration(TRANSITION_DURATION)
        .attr('x', function(d) {

            return SCALES.x(xform(d - 0.5, lap));
        })
        .attr('width', function(d) {

            return SCALES.x(xform(d + 0.5, lap)) - SCALES.x(xform(d - 0.5, lap));
        });

    // Transition lapped elements.
    vis.selectAll('rect.lapped')
        .transition()
        .duration(TRANSITION_DURATION)
        .attr('x', function(d, i) {

            return SCALES.x(xform(i + 0.5, lap));
        })
        .attr('width', function(d, i) {

            return SCALES.x(xform(i + 1.5, lap)) - SCALES.x(xform(i + 0.5, lap));
        });

    // Transition lapped elements.
    vis.selectAll('polyline.placing')
        .transition()
        .duration(TRANSITION_DURATION)
        .attr('points', function(d) {

            var points = [];
            for (var i = 0;
                 i < d.placing.length;
                 i++) {

                points[i] = SCALES.x(xform(i, lap)) + ',' + SCALES.y(d.placing[i] - 1);
            }

            if (points.length > 0) {

                points.push(SCALES.x(xform(i - 0.5, lap)) + ',' + SCALES.y(d.placing[i - 1] - 1));
            }

            return points.join(' ');
        });

    // Transition markers (circles).
    vis.selectAll('circle.marker')
        .transition()
        .duration(TRANSITION_DURATION)
        .attr('cx', function(d) {

            return SCALES.x(xform(d.lap, lap));
        });

    // Transition markers (labels).
    vis.selectAll('text.label.marker')
        .transition()
        .duration(TRANSITION_DURATION)
        .attr('x', function(d) {

            return SCALES.x(xform(d.lap, lap));
        });
}

/**
 * The zooming function is piecewise linear.  It divides the x-axis into several sections each of which is zoomed by
 * a different amount.  The closer the zone is to the zoom centre, the higher the zoom factor.
 *
 * | NO ZOOM | ZOOM_SHOULDER | ZOOM_PEAK | ZOOM_SHOULDER | NO ZOOM |
 *
 * ZOOM_PEAK is applied where on the lap where the user clicked.
 * ZOOM_SHOULDER is applied to the laps either side of this.
 * No zoom is applied elsewhere.
 *
 * @param x the x-coordinate to transform using the zooming function.
 * @param lap the lap the user clicked on.
 */
function zoomXform(x, lap) {

    // The x-axis domain.
    var domain = SCALES.x.domain();
    var step = domain[1] - domain[0];

    // What is the increment between each lap after zooming.
    var inc = lap <= domain[0] || lap >= domain[1] ?
        step / (ZOOM_PEAK + ZOOM_SHOULDER - 2.0 + step) :
        step / (ZOOM_PEAK + 2.0 * ZOOM_SHOULDER - 3.0 + step);

    // The zoom centre is mid-lap.
    lap += 0.5;

    // The transformed version of x.
    var z = 0.0;

    // Beyond upper shoulder.
    if (x > lap + 1.0) z = (x + ZOOM_PEAK + 2.0 * ZOOM_SHOULDER - 3.0) * inc;

    // Upper shoulder.
    else if (x > lap) z = ((x - lap + 1.0) * ZOOM_SHOULDER + lap + ZOOM_PEAK - 2.0) * inc;

    // Peak.
    else if (x > lap - 1.0) z = ((x - lap + 1.0) * ZOOM_PEAK + lap + ZOOM_SHOULDER - 2.0) * inc;

    // Lower shoulder.
    else if (x > lap - 2.0)   z = ((x - lap + 2.0) * ZOOM_SHOULDER + lap - 2.0) * inc;

    // Below lower shoulder.
    else z = (x - domain[0]) * inc;

    return z;
}

function unzoomXform(x) {

    return x;
}

// Add safety car laps (rectangle elements).
//
// vis: the data visualization root.
// data: safety car laps.
//
function addSafetyElement(vis, data) {

    if (data != undefined) {

        var y = SCALES.y.range()[0];
        var height = SCALES.y.range()[1] - y;
        var width = SCALES.x(1) - SCALES.x(0);

        vis.selectAll('rect.safety')
            .data(data)
            .enter()
            .append('svg:rect')
            .attr('class', 'safety zoom')
            .attr('x', function(d) {

                return SCALES.x(d - 0.5);
            })
            .attr('y', function() {

                return y;
            })
            .attr('height', function() {

                return height;
            })
            .attr('width', function() {

                return width;
            });
    }
}

// Add lapped rectangle elements.
//
// vis: the data visualization root.
// data: the lapped data.
//
function addLappedElement(vis, data) {

    if (data != undefined) {

        var width = SCALES.x(1) - SCALES.x(0);

        vis.selectAll('rect.lapped')
            .data(data)
            .enter()
            .append('svg:rect')
            .attr('class', 'lapped zoom')
            .attr('x', function(d, i) {

                return SCALES.x(i + 0.5);
            })
            .attr('y', function(d) {

                return SCALES.y(d > 0 ? d - 1.5 : 0);
            })
            .attr('height', function(d) {

                return d > 0 ? SCALES.y.range()[1] - SCALES.y(d - 1.5) : 0;
            })
            .attr('width', function(d) {

                return d > 0 ? width : 0;
            });
    }
}
// Add lap tick-lines.
//
// vis: the data visualization root.
// lapCount: number of laps.
//
function addLapTickLines(vis, lapCount) {

    vis.selectAll('line.tickLine')
        .data(SCALES.x.ticks(lapCount))
        .enter().append('svg:line')
        .attr('class', 'tickLine zoom')
        .attr('x1', function(d) {
            return SCALES.x(d);
        })
        .attr('x2', function(d) {
            return SCALES.x(d);
        })
        .attr('y1', SCALES.y.range()[0] - TICK_MARK_LENGTH)
        .attr('y2', SCALES.y.range()[1] + TICK_MARK_LENGTH)
        .attr('visibility', function(d) {

            return d <= lapCount ? 'visible' : 'hidden'
        });
}

// Add lap labels.
//
// vis: the data visualization root.
// data: lap data.
// y: y position of labels.
// dy: y offset.
// cssClass: CSS class id.
//
function addLapLabels(vis, data, y, dy, cssClass) {

    vis.selectAll('text.lap.' + cssClass)
        .data(SCALES.x.ticks(data))
        .enter().append('svg:text')
        .attr('class', 'lap ' + cssClass + ' zoom')
        .attr('x', function(d) {

            return SCALES.x(d);
        })
        .attr('y', y)
        .attr('dy', dy)
        .attr('text-anchor', 'middle')
        .text(function(d, i) {

            return i > 0 ? i : '';
        });
}

// Add placings polyline elements.
//
// vis: the visualization root.
// laps: lap data.
//
function addPlacingsLines(vis, laps) {

    vis.selectAll('polyline.placing')
        .data(laps)
        .enter()
        .append('svg:polyline')
        .attr('class', 'placing zoom')
        .attr('points', function(d) {

            var points = [];
            for (var i = 0;
                 i < d.placing.length;
                 i++) {

                points[i] = SCALES.x(i) + ',' + SCALES.y(d.placing[i] - 1);
            }

            if (points.length > 0)
                points.push(SCALES.x(i - 0.5) + ',' + SCALES.y(d.placing[i - 1] - 1));

            return points.join(' ');
        })
        .style('stroke', function(d) {

             return (vis, d.color);
        })
        .on('mouseover', function(d) {

            highlight(vis, d.name);

        })
        .on('mouseout', function() {

            unhighlight(vis);
        })
        ;
}

// Add driver name labels.
//
// vis: the data visualization root.
// laps: the lap data.
// cssClass: CSS class id.
// textAnchor: text-anchor value.
//
function addDriverLabels(vis, laps, cssClass, x, textAnchor) {

    return vis.selectAll('text.label.' + cssClass)
        .data(laps)
        .enter()
        .append('svg:text')
        .attr('class', 'label ' + cssClass)
        .attr('x', x)
        .attr('dy', '0.05em')
        .attr('text-anchor', textAnchor)
        .text(function(d) {

            return d.name;
        })
        .style('fill', function(d) {

//            return "rgb(69, 69, 69)";
            return (vis, d.color);
        })
        .on('mouseover', function(d) {

            highlight(vis, d.name);
        })
        .on('mouseout', function() {

            unhighlight(vis);

        });
}

// Add markers.
//
// vis: the visualization root.
// data: marker data.
// class: marker sub-class.
// label: marker label.
//
function addMarkers(vis, data, cssClass, label) {
    label = label || "P";

    // Place circle glyph.
    vis.selectAll("circle.marker." + cssClass)
        .data(data)
        .enter()
        .append("svg:circle")
        .attr("class", "marker " + cssClass + " zoom")
        .attr("cx", function(d) {

            return SCALES.x(d.lap);
        })
        .attr("cy", function(d) {

            return SCALES.y(d.placing - 1);
        })
        .attr("r", MARKER_RADIUS)
        .style("fill", function(d) {
			//return color;
            return SCALES.clr(d.start);

        })
        .on('mouseover', function(d) {

            highlight(vis, d.name);
        })
        .on('mouseout', function() {

            unhighlight(vis);
        });

    // Place text.
    vis.selectAll("text.label.marker." + cssClass)
        .data(data)
        .enter()
        .append("svg:text")
        .attr("class", "label marker " + cssClass + " zoom")
        .attr("x", function(d) {

            return SCALES.x(d.lap);
        })
        .attr("y", function(d) {

            return SCALES.y(d.placing - 1);
        })
        .attr("dy", "0.35em")
        .attr("text-anchor", "middle")
        .text(label)
        .on('mouseover', function(d) {

            highlight(vis, d.name);
        })
        .on('mouseout', function() {

            unhighlight(vis);
        });
}

// Gets the window dimensions.
//
function getWindowDimensions() {

    var width = 630;
    var height = 460;
    if (document.body && document.body.offsetWidth) {

        width = document.body.offsetWidth;
        height = document.body.offsetHeight;
    }

    if (document.compatMode == 'CSS1Compat' && document.documentElement && document.documentElement.offsetWidth) {

        width = document.documentElement.offsetWidth;
        height = document.documentElement.offsetHeight;
    }

    if (window.innerWidth && window.innerHeight) {

        width = window.innerWidth;
        height = window.innerHeight;
    }

    return {'width': width, 'height': height};
}
</script>

<div id="chart"></div>

How will things turn out in next season?
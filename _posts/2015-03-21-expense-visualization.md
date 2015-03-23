---
layout: post
title:  "Expense Visualization"
date:   2015-02-01 14:22:17
categories: visualization
img:   public/img/cambodia-expenses.png
tags:
- d3.js
- Google Sheets
- visualization
- responsive design
---


The first leg of [my travels][castaway-life] is over. Money is tight, so naturally I was curious to compare the costs of the various countries I visited.  I also wondered if I'd come close to my budget, or just blown past it.

This analysis could have been handled entirely in Google Sheets. Google Sheets [^1] is where I kept track of the expenses so that would be entirely natural.

But built-in charts aren't exactly what I wanted. I wanted more control over the presentation and some specific interactions designs. Besides, I wanted to refresh my skills after months away from code so the cost in time of this extravagance was worth it.

So it was that I created some `d3.js` visualizations of my expense data and published them on the web. Here's now.

### Design

The web page needed to import the expense data from Google Sheets and render it in charts.  The most salient information is:

1. Total spend and average spend per day, per category and per country
2. Top expenses
3. Trend of those expenses over time

#### Importing from Google Sheets

There are a couple of options for importing data from Google Sheets. I wanted the solution to be live-updating so I could use this page in future travels for daily monitoring.  Any export of the data was thus ruled out.

If you make the sheet public, you can use a client-side approach to querying the data via JSON, as described [here][public-sheet-query-via-json].  This is a cool technique and I almost used it, but I didn't want this data to be public in its raw form.  Not on any 'real' grounds — I'm not fearful of the security implications of the world knowing by spending habits — but rather as a matter of principle: I'd rather design something more general and in general one wouldn't want to share all the raw data all the time.

So that left a server-side solution, something to authenticate in private and then proxy the data over to the client.  I found [this][accessing-google-spreadsheets-from-node-js] excellent writeup on how to do this from node.js, which I've been meaning to play with some more, so I ended up implementing that solution pretty much wholesale.

As is typical of any visualization project, the first and lamest part of the work is mangling the data from its original format into one you can use.  In the case of this project, Google Sheets provides

```
Object {rows: Object, info: Object}

data.rows[1]
Object {1: "Country", 2: "Date", 3: "Cost", 4: "Cash Cost", 5: "Currency", 6: "Item", 7: "Category", 8: "Joint Cost (I)", 9: "Currency", 10: "Joint Cost (M)", 11: "Currency"}

data.rows[3]
Object {1: "Cambodia", 2: "2/16/2015", 3: "=D3*VLOOKUP(E3, ExchangeRate!R3C1:R11C2 ,2,FALSE)", 4: 30, 5: "USD", 6: "Cambodia tourist visa", 7: "Miscellaneous"}
```

These hashmaps don't work so well with `d3.js`'s array-focused data formats.

So the munging code takes the hashmaps, finds the header row then generates all the data rows as an array of hashmaps of `category : value`.

```javascript
> JSON.stringify(chart_data.raw[0])
"{"Country":"Cambodia",
"Date":"2015-02-16T08:00:00.000Z",
"Cash Cost":30,
"Currency":"USD",
"Item":"Cambodia tourist visa",
"Category":"Miscellaneous",
"y":30,
"x":"2015-02-16T08:00:00.000Z",
"category_idx":0,
"detail":"Cambodia tourist visa — $30.00"}"
```

<aside>
When munging the question is always whether to do it at the client or server side.  My experience has been it's better to do it client side.  Of course remove any truly weird structures on the server, but in general just transparently proxy the data from its source (e.g. a SQL server) and let the client code transform it for rendering.  Otherwise debugging becomes too messy with bugs at both ends to contend with.
</aside>

#### Server Side

Now the server design was party clear: a node.js JSON endpoint to serve the data.  Since I was already running the `express` web server, it was a simple matter to include a separate port to serve some static HTML pages for the client-side bits.

The module `travel-data.js` loads the spreadsheet and exports the data as `budget_info`.  Then we set up two routes in express, `/` for the static site and `/data` for the JSON data.

```javascript
var express = require("express"),
    travel_data = require("./travel-data.js"),
    app = express(),
    bodyParser = require('body-parser'),
    errorHandler = require('errorhandler'),
    methodOverride = require('method-override');
    
exports.publicDir = process.argv[2] || __dirname + '/public';

app.get("/", function (req, res) {
  res.redirect("/index.html");
});

app.get("/data", function (req, res) {
    var store = '';

    res.setHeader("Content-Type", "text/json");
    res.setHeader("Access-Control-Allow-Origin", "*");
    res.end(JSON.stringify(travel_data.budget_info))
});

app.use(methodOverride());
app.use(bodyParser.json());
app.use(bodyParser.urlencoded({
  extended: true
}));
app.use(express.static(exports.publicDir));
app.use(errorHandler({
  dumpExceptions: true,
  showStack: true
}));

exports.app = app;
```

#### Client Side Design

To visualize the data, I settled on two views: a daily bar chart with lines for the moving averages of the spends in each category and a top expenses.  These two views operate on one country at a time, selected by an option box.  In the future there could easily be a "all" countries mode which shows the views for all the data.

Switching into the "top expenses" mode is accomplished with a button; you get out of it by clicking anywhere on the page.  This turned out to be much more natural than a toggle button or two buttons, especially on touch interfaces.  Essentially this approach is a modal dialog box.

To give some visual cueing to tie the two views together, I decided to transition the bars representing the expenses from the bar chart to the stacked list of bars representing the top expenses.  So the biggest expenses fly out from their homes in the bar chart and stack in the center of the screen with their descriptions.

#### Responsiveness

Any sane web development today needs to work perfectly on mobile devices. I'd rather argue the side which says everything should mobile-only rather than desktop-only.  So this visualization needed to be responsive.

The easiest option is the default: `d3.js` reders to SVG, which is itself scalable.  So just do nothing and users can resize the drawing at will.

There are problems with this approach.  Visual elements may fit on the screen and be perfectly rendered but they may also be tiny.  Too tiny to read comfortably.  The article ["Building Responsive Visualizations in d3.js"][building-responsive-visualizations-d3-js] elaborates on these problems and provides a solution:

1. Re-render the chart upon resize
    * Add or remove tick marks based on screen size
    * Add or remove datapoints based on screen size; rendering detail beyond the pixel level is wasteful and can also make the chart look "thick".
2. When the viewport gets too small, switch to sparklines to minimize clutter

I add to this a third stage between the two: when the viewport gets to small, remove the bar charts but keep the axes and the lines.

This approach entails moving a bunch of the geometry-specific code to a "`resize()`" function which gets called when the containing element changes size.  `resize()` can then make a bunch of decisions about what elements to render based on the size of he viewport.

##### Resize() notes

In the example from the article, the new elements were re-rendered on resize by updating the scale and then calling the helper `d3.js` objects/functions to create/update the SVG elements:

```javacript
/* Update the axis with the new scale */
graph.select('.x.axis')
  .attr("transform", "translate(0," + height + ")")
  .call(xAxis);
 
graph.select('.y.axis')
  .call(yAxis);
 
/* Force D3 to recalculate and update the line */
graph.selectAll('.line')
  .attr("d", line);
```

This is great, but what do you do if you're making bar charts or rectangles?  When there's no canned `d3.svg.axis()` or `d3.svg.line()` to simply generate the SVG.

The solution I used was to split out the geometry attribute setting into separate functions, which then get `call`'ed in the resize function:


```javascript
    var topbars = function(g) {
            g
                .attr("width", (xScale(1)-xScale(0)) * 0.80 )
                .attr("y", function(d) {
                        console.assert( !isNaN(yScale(bar_stack_y0[d.category_idx][d.x])));
                        return yScale(bar_stack_y0[d.category_idx][d.x]+d.y); })
                .attr("x", function(d) { return xScale(d.x); })
                .attr("height", function(d) { return Math.abs(yScale(0) - yScale(d.y)); });
    }
    
    var seriesLabel = function(g) {
        g.each(function() {
            var g = d3.select(this);
            g
                .attr("x", function(d) { return xScale(d.x); })
                .attr("y", function(d) { return yScale(d.y + d.y0 + d.y_offset); });
        } );
    };

    var titleSeriesLabel = function(g) {
        g.each(function() {
            var g = d3.select(this);
            g
                .attr("x", function(d) { return xScale(d.x); })
                .attr("y", function(d) { return yScale(yScale.domain()[1] + yScale.invert(0) - yScale.invert(0)); });
        } );
    };
```

In the end though, the right design would be to save all geometry setting for the resize function, and make all visual elements rendered before `resize()` be invisible in the initial setup. 

#### Nuances

A few small visual nuances help make the rendering more pleasant:

1. The bar chart is animated so the expenses grow from 0.  This is a nice transition into the chart.
1. In the "Top Expenses" mode, the background is blended, giving it a frosted-glass look to keep it in the viewer's mind but minimize distractions.
1. I originally wanted a moving average line for each expense category leading to the all-dates average at the right-hand side of the hard.  There is no built-in moving average interpolation in `d3.js`, so I had to write one, cribbing heavily from [this article][plot-rolling-moving-average-in-d3-js].
1. I wanted to have the moving average series lines culminate in a label for the series. This makes more sense to be than the typical 'legend box' but was problematic: if the averages are close (e.g. if the responsive design causes them the average lines to be just a few pixels apart), the labels will overlap.
    * My first attempt at a solution was to try to be clever: use `d3.js`'s [force-directed layout to lay the labels out][force-layout-of-d3-labels].  The step function would constrain the `x` coordinate to stay put, leaving the labels to move gracefully away from each other along a vertical line.  This worked, but the effect of the labels bouncing around at page load time was unplesant.
    * The second attempt was uglier.  Query the sizes of the labels and offset them if they overlap.  This was faster, conceptually simpler, but made for messy code.  The visual effect was better however so that's the one I went with.
    
### Visualization

That's it.  The visualization is [here][cost-viz-app].  Here's what it looks like:

<img style="width: 100%;" src="http://www.thecastawaylife.com/blog/wp-content/uploads/2015/03/expensesthailand.png" alt="Thailand Expenses" title="thailand.png"/>

<img style="width: 100%;" src="http://www.thecastawaylife.com/blog/wp-content/uploads/2015/03/expensescambodia-top-expenses.png" alt="Cambodia top expenses" title="cambodia-top-expenses.png"/>

There are a series of insights from the visualizations [here][castaway-life-expenses].

[force-layout-of-d3-labels]: https://stackoverflow.com/questions/17425268/d3js-automatic-labels-placement-to-avoid-overlaps-force-repulsion
[public-sheet-query-via-json]: https://coderwall.com/p/duapqq/use-a-google-spreadsheet-as-your-json-backend
[accessing-google-spreadsheets-from-node-js]: http://www.nczonline.net/blog/2014/03/04/accessing-google-spreadsheets-from-node-js/
[building-responsive-visualizations-d3-js]: https://blog.safaribooksonline.com/2014/02/17/building-responsible-visualizations-d3-js/
[plot-rolling-moving-average-in-d3-js]: https://stackoverflow.com/questions/11963352/plot-rolling-moving-average-in-d3-js

[cost-viz-app]: https://tranquil-river-4676.herokuapp.com/index.html
[castaway-life]: http://thecastawaylife.com/blog
[castaway-life-expenses]:  http://www.thecastawaylife.com/blog/?p=829

[resize-svg-when-window-is-resized-in-d3-js]: https://stackoverflow.com/questions/16265123/resize-svg-when-window-is-resized-in-d3-js
[proper-way-to-return-json-using-node-or-express]: https://stackoverflow.com/questions/19696240/proper-way-to-return-json-using-node-or-express



[^1]: Why Google sheets and not Mint? Most of the places I visited were cash-only and used multiple currencies.  If was easier to track that it in Sheets

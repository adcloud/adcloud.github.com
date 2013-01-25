---
layout: page
title: "Technologies"
comments: false
sharing: true
footer: true
---

<script src="/javascripts/libs/d3.js"></script>
<script src="/javascripts/libs/d3.layout.cloud.js"></script>
<svg></svg>

<script>
  var fill = d3.scale.category20();

  d3.layout.cloud().size([600, 600])
      .words([
        "MySQL",
"Redis",
"Riak",
"CouchDB",
"Hadoop",
"Memcached",
"MongoDB",
"S3",
"PHP",
"Ruby",
"JavaScript",
"Python",
"Clojure",
"Django",
"Ruby on Rails",
"node.js",
"Backbone",
"symfony",
"CakePHP",
"Chef",
"AWS",
"EC2",
"DynamoDB",
"RabbitMQ",
"Github:E",
"Statsd",
"Graphite",
"Icinga",
"Munin"
].map(function(d) {
        return {text: d, size: 10 + Math.random() * 90};
      }))
      .rotate(function() { return ~~(Math.random() * 2) * 90; })
      .font("Impact")
      .fontSize(function(d) { return d.size; })
      .on("end", draw)
      .start();

  function draw(words) {
    d3.select("svg")
        .attr("width", 600)
        .attr("height", 600)
      .append("g")
        .attr("transform", "translate(300,300)")
      .selectAll("text")
        .data(words)
      .enter().append("text")
        .style("font-size", function(d) { return d.size + "px"; })
        .style("font-family", "Impact")
        .style("fill", function(d, i) { return fill(i); })
        .attr("text-anchor", "middle")
        .attr("transform", function(d) {
          return "translate(" + [d.x, d.y] + ")rotate(" + d.rotate + ")";
        })
        .text(function(d) { return d.text; });
  }
</script>
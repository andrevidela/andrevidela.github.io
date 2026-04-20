---
layout: page
title: Stellar
description: A library for API programming
img: assets/img/stellar-logo.png
importance: 1
category: research
related_publications: false
---

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/stellar-logo-large.png" title="example image" class="img-fluid rounded z-depth-1" %}
    </div>
</div>


[Stellar](https://gitlab.com/avidela/stellar) is an Idris2 library based on containers as APIs. It provides tools to write software at a very high level
by manipulating APIs first and then filling the program that translate between APIs. Currently, it offers functionality to run sqlite3 databases,
HTTP requests from Node, HTTP requests from the browser, and command line interfaces.

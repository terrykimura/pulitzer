---
layout: page
datasource: export_fiction
---

{% for winner in site.data[ page.datasource ] %}
{{ winner.year }}: <i>{{ winner.title }}</i> by {{ winner.author }}
{% endfor %}
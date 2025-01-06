---
layout: page
datasource: export_generalnonfiction
---

{% for winner in site.data[ page.datasource ] %}
{{ winner.year }}: <i>{{ winner.title }}</i> by {{ winner.author }}
{% endfor %}
---
layout: default
title: "Via ferrata"
permalink: /via-ferrata/
---

<nav style="margin:8px 0 16px 0;">
  <a href="{{ '/' | relative_url }}">Randonnées</a> ·
  <strong>Via ferrata</strong>
</nav>

<h1>Via ferrata</h1>

<input id="search" type="text" placeholder="Rechercher (titre, tags, lieu...)" style="padding:8px;width:100%;max-width:480px;margin:12px 0;" />

<ul id="cards" style="list-style:none;padding:0;display:grid;gap:12px;">
  {% assign vf = site.posts | where_exp: 'p','p.categories contains "via-ferrata"' | sort: 'date' | reverse %}
  {% for p in vf %}
  <li class="card"
      data-title="{{ p.title | downcase }}"
      data-tags="{{ p.tags | join:' ' | downcase }}"
      style="border:1px solid #e5e7eb;border-radius:14px;padding:14px;">
    <a href="{{ p.url | relative_url }}" style="text-decoration:none;display:block;margin:0 0 4px 0;">
      <strong style="font-size:1.1rem;">{{ p.title | strip_html }}</strong>
    </a>
    <div style="font-size:14px;color:#555;">
      {{ p.date | date: "%d %b %Y" }}
      {% if p.difficulte %} — {{ p.difficulte }}{% endif %}
      {% if p.cotation %} • Cotation: {{ p.cotation }}{% endif %}
      {% if p.lieu %} • {{ p.lieu }}{% endif %}
    </div>
    {% if p.tags %}
      <div style="margin-top:6px;">
        {% for t in p.tags %}
          <span style="border:1px solid #ddd;border-radius:10px;padding:2px 8px;margin-right:6px;font-size:12px;">#{{ t }}</span>
        {% endfor %}
      </div>
    {% endif %}
  </li>
  {% endfor %}
</ul>

<script>
const q = document.getElementById('search');
const cards = [...document.querySelectorAll('.card')];
function applyFilters(){
  const text = (q.value || '').toLowerCase().trim();
  cards.forEach(c => {
    let ok = true;
    if(text){
      const hay = (c.dataset.title + ' ' + c.dataset.tags).toLowerCase();
      ok = hay.includes(text);
    }
    c.style.display = ok ? '' : 'none';
  });
}
q.addEventListener('input', applyFilters);
</script>

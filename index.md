---
layout: default
title: "Toutes mes randonnées"
---

<h1>Toutes mes randonnées</h1>

<input id="search" type="text" placeholder="Rechercher (titre, tags, lieu...)" style="padding:8px;width:100%;max-width:480px;margin:12px 0;" />

<div style="margin:8px 0;">
  <strong>Filtres rapides :</strong>
  <label><input type="checkbox" class="filter" data-key="difficulte" value="Facile"> Facile</label>
  <label><input type="checkbox" class="filter" data-key="difficulte" value="Moyenne"> Moyenne</label>
  <label><input type="checkbox" class="filter" data-key="difficulte" value="Difficile"> Difficile</label>
</div>

<ul id="cards" style="list-style:none;padding:0;display:grid;gap:12px;">
  {% assign sorted = site.posts | sort: 'date' | reverse %}
  {% for p in sorted %}
  <li class="card"
      data-title="{{ p.title | downcase }}"
      data-tags="{{ p.tags | join:' ' | downcase }}"
      data-difficulte="{{ p.difficulte }}"
      style="border:1px solid #e5e7eb;border-radius:14px;padding:14px;">
      <a href="{{ p.url | relative_url }}" style="text-decoration:none;display:block;margin:0 0 4px 0;">
      <strong style="font-size:1.1rem;">{{ p.title | strip_html }}</strong>
      </a>
    <div style="font-size:14px;color:#555;">
      {{ p.date | date: "%d %b %Y" }} — {{ p.distance }} • D+ {{ p.denivele }} • {{ p.difficulte }}
      {% if p.lieu %} • {{ p.lieu }}{% endif %}
    </div>
    {% if p.tags %}<div style="margin-top:6px;">{% for t in p.tags %}<span style="border:1px solid #ddd;border-radius:10px;padding:2px 8px;margin-right:6px;font-size:12px;">#{{ t }}</span>{% endfor %}</div>{% endif %}
  </li>
  {% endfor %}
</ul>

<script>
const q = document.getElementById('search');
const cards = [...document.querySelectorAll('.card')];
const filters = [...document.querySelectorAll('.filter')];

function applyFilters(){
  const text = (q.value || '').toLowerCase().trim();
  const active = filters.filter(f => f.checked).map(f => ({key:f.dataset.key, val:f.value}));
  cards.forEach(c => {
    let ok = true;
    if(text){
      const hay = (c.dataset.title + ' ' + c.dataset.tags).toLowerCase();
      ok = hay.includes(text);
    }
    if(ok && active.length){
      for(const f of active){
        if((c.dataset[f.key] || '') !== f.val){ ok = false; break; }
      }
    }
    c.style.display = ok ? '' : 'none';
  });
}
q.addEventListener('input', applyFilters);
filters.forEach(f => f.addEventListener('change', applyFilters));
</script>

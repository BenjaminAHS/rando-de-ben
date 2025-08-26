---
layout: default
title: "Randonnées"
---

<h1>Toutes mes aventures</h1>


<nav style="margin:8px 0 16px;">
  {% assign home_url = '/' | relative_url %}
  {% assign vf_url = '/via-ferrata/' | relative_url %}

  <a href="{{ home_url }}"
     {% if page.url == home_url %}style="font-weight:700;" aria-current="page"{% endif %}>
     Randonnées
  </a>
  ·
  <a href="{{ vf_url }}"
     {% if page.url == vf_url %}style="font-weight:700;" aria-current="page"{% endif %}>
     Via ferrata
  </a>
</nav>

<!-- ====== CARTE DES SORTIES ====== -->
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
<div id="mapIndex" style="height:520px;border-radius:12px;margin:12px 0;"></div>
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<!-- GPX -> GeoJSON pour calcul du point si lat/lng absents -->
<script src="https://cdn.jsdelivr.net/npm/@mapbox/togeojson@0.16.0/dist/togeojson.umd.js"></script>

<script>
  const SITE = '{{ site.url }}{{ site.baseurl }}';

  // Données générées par Jekyll depuis tes posts
  const items = [
  {% assign posts_sorted = site.posts | sort: 'date' | reverse %}
  {% for p in posts_sorted %}
    {
      title: {{ p.title | jsonify }},
      url: "{{ p.url | relative_url }}",
      type: "{% if p.categories contains 'via-ferrata' %}via{% else %}rando{% endif %}",
      lat: {% if p.lat %}{{ p.lat }}{% else %}null{% endif %},
      lng: {% if p.lng %}{{ p.lng }}{% else %}null{% endif %},
      gpx: {% if p.gpx %}"{{ site.url }}{{ site.baseurl }}{{ p.gpx }}?v={{ site.time | date: '%s' }}"{% else %}null{% endif %}
    }{% unless forloop.last %},{% endunless %}
  {% endfor %}
  ];

  // Carte
  const map = L.map('mapIndex');
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {attribution:'© OpenStreetMap'}).addTo(map);
  map.setView([46.5, 2.5], 6); // France par défaut

  const group = L.featureGroup().addTo(map);

  // Style marqueurs (rando vs via)
  function styleFor(type){
    return type === 'via'
      ? {radius:7, weight:1.5, color:'#b91c1c', fillColor:'#ef4444', fillOpacity:0.8} // via ferrata (rouge)
      : {radius:7, weight:1.5, color:'#1d4ed8', fillColor:'#3b82f6', fillOpacity:0.8}; // rando (bleu)
  }

  // Ajoute un point (lat/lng connus)
  function addPoint(it, lat, lng){
    const marker = L.circleMarker([lat, lng], styleFor(it.type))
      .bindPopup(`<b>${it.title}</b><br><a href="${it.url}">Voir la fiche</a>`);
    marker.addTo(group);
  }

  // Si pas de lat/lng, calcule via GPX (centre des bounds)
  function addFromGpx(it){
    if(!it.gpx) return;
    return fetch(it.gpx)
      .then(r => r.text())
      .then(txt => {
        const dom = new DOMParser().parseFromString(txt, 'application/xml');
        const gj = toGeoJSON.gpx(dom);
        if (!gj || !gj.features || !gj.features.length) return;
        const layer = L.geoJSON(gj);
        const b = layer.getBounds();
        const c = b.getCenter();
        addPoint(it, c.lat, c.lng);
      })
      .catch(()=>{ /* silencieux si une trace ne charge pas */ });
  }

  // Pipeline : pose les points
  const pending = [];
  items.forEach(it => {
    if (it.lat !== null && it.lng !== null) {
      addPoint(it, it.lat, it.lng);
    } else if (it.gpx) {
      pending.push(addFromGpx(it));
    }
  });

  // Ajuste la vue après ajout (y compris async GPX)
  Promise.all(pending).then(() => {
    if (group.getLayers().length) {
      map.fitBounds(group.getBounds().pad(0.1));
    }
  });

  // Petite légende
  const legend = L.control({position:'topright'});
  legend.onAdd = function(){
    const d = L.DomUtil.create('div','legend');
    d.style.background = 'white'; d.style.padding='6px 8px'; d.style.borderRadius='8px'; d.style.boxShadow='0 1px 4px rgba(0,0,0,.1)';
    d.innerHTML = `
      <div style="display:flex;gap:10px;align-items:center;">
        <span style="display:inline-block;width:10px;height:10px;background:#3b82f6;border:2px solid #1d4ed8;border-radius:50%;"></span> Randonnée
      </div>
      <div style="display:flex;gap:10px;align-items:center;margin-top:6px;">
        <span style="display:inline-block;width:10px;height:10px;background:#ef4444;border:2px solid #b91c1c;border-radius:50%;"></span> Via ferrata
      </div>`;
    return d;
  };
  legend.addTo(map);
</script>
<!-- ====== FIN CARTE ====== -->


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

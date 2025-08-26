---
layout: default
title: "Mes randonnées"
---

<!-- Menu -->
<nav style="margin:8px 0 16px;">
  <a href="{{ '/' | relative_url }}">Randonnées</a> ·
  <a href="{{ '/via-ferrata/' | relative_url }}">Via ferrata</a>
</nav>

<!-- ====== CARTE DES SORTIES ====== -->
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
<div id="mapIndex" style="height:520px;border-radius:12px;margin:12px 0;"></div>
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mapbox/togeojson@0.16.0/dist/togeojson.umd.js"></script>
<script src="https://unpkg.com/leaflet-omnivore@0.3.4/leaflet-omnivore.min.js"></script>

<script>
  // --- utils robustes ---
  const toNum = v => {
    if (v === null || v === undefined) return null;
    const n = Number(String(v).trim().replace(',', '.'));
    return Number.isFinite(n) ? n : null;
  };

  // Carte (vue France par défaut — on ne recentre pas automatiquement)
  const map = L.map('mapIndex');
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', { attribution:'© OpenStreetMap' }).addTo(map);
  map.setView([46.5, 2.5], 6);

  const group = L.featureGroup().addTo(map);
  const fails = [];

  function styleFor(type){
    return type === 'via'
      ? {radius:7, weight:1.5, color:'#b91c1c', fillColor:'#ef4444', fillOpacity:0.8}
      : {radius:7, weight:1.5, color:'#1d4ed8', fillColor:'#3b82f6', fillOpacity:0.8};
  }
  function addPoint(it, lat, lng){
    L.circleMarker([lat, lng], styleFor(it.type))
     .bindPopup(`<b>${it.title}</b><br><a href="${it.url}">Voir la fiche</a>`)
     .addTo(group);
  }

  // Posts -> tableau d'items (lat/lng ou centre de GPX)
  const items = [
  {% assign posts_sorted = site.posts | sort: 'date' | reverse %}
  {% for p in posts_sorted %}
    {
      title: {{ p.title | jsonify }},
      url: "{{ p.url | relative_url }}",
      type: "{% if p.categories contains 'via-ferrata' %}via{% else %}rando{% endif %}",
      lat: {% if p.lat %}{{ p.lat }}{% elsif p.latitude %}{{ p.latitude }}{% else %}null{% endif %},
      lng: {% if p.lng %}{{ p.lng }}{% elsif p.lon %}{{ p.lon }}{% elsif p.longitude %}{{ p.longitude }}{% else %}null{% endif %},
      gpx: {% if p.gpx %}"{{ site.url }}{{ site.baseurl }}{{ p.gpx }}?v={{ site.time | date: '%s' }}"{% else %}null{% endif %}
    }{% unless forloop.last %},{% endunless %}
  {% endfor %}
  ];

  function addFromGpx(it){
    if(!it.gpx) return Promise.resolve();
    // 1) Essai via toGeoJSON (léger)
    return fetch(it.gpx)
      .then(r => { if(!r.ok) throw new Error('HTTP '+r.status); return r.text(); })
      .then(txt => {
        if (!window.toGeoJSON) throw new Error('toGeoJSON indisponible');
        const dom = new DOMParser().parseFromString(txt, 'application/xml');
        if (dom.getElementsByTagName('parsererror').length) throw new Error('XML invalide');
        const gj = toGeoJSON.gpx(dom);
        if (!gj || !gj.features || !gj.features.length) throw new Error('Aucune feature');
        const b = L.geoJSON(gj).getBounds().getCenter();
        addPoint(it, b.lat, b.lng);
      })
      .catch(() => {
        // 2) Fallback via omnivore (robuste)
        return new Promise(resolve => {
          if (!window.omnivore) { fails.push(it.title + ' (lib manquante)'); return resolve(); }
          const temp = omnivore.gpx(it.gpx)
            .on('ready', function(){
              const c = this.getBounds().getCenter();
              addPoint(it, c.lat, c.lng);
              map.removeLayer(this); // on n'affiche pas la trace sur l'accueil
              resolve();
            })
            .on('error', function(){
              fails.push(it.title);
              resolve();
            })
            .addTo(map);
        });
      });
  }

  const pending = [];
  items.forEach(it => {
    const lat = toNum(it.lat);
    const lng = toNum(it.lng);
    if (lat !== null && lng !== null) {
      addPoint(it, lat, lng);
    } else if (it.gpx) {
      pending.push(addFromGpx(it));
    } else {
      fails.push(it.title + ' (pas de lat/lng ni gpx)');
    }
  });

  // On NE recentre PAS automatiquement (on garde la vue France)
  Promise.all(pending).then(() => {
    if (!group.getLayers().length) {
      const p = document.createElement('p');
      p.style.color = 'crimson';
      p.textContent = 'Aucun point placé : ajoute lat/lng (45.2723 / 5.7763) ou un GPX.';
      document.getElementById('mapIndex').after(p);
    }
    if (fails.length){
      const p = document.createElement('p');
      p.style.color = '#a16207';
      p.textContent = 'Non placés (coordonnées/GPX non lus) : ' + fails.join(', ');
      document.getElementById('mapIndex').after(p);
    }
    console.log('Markers placés:', group.getLayers().length);
  });

  // Bouton "Voir toutes les sorties"
  const zoomCtrl = L.control({position:'topleft'});
  zoomCtrl.onAdd = function(){
    const btn = L.DomUtil.create('button');
    btn.textContent = 'Voir toutes les sorties';
    btn.style.cssText = 'background:white;border:1px solid #ddd;border-radius:8px;padding:6px 10px;cursor:pointer;box-shadow:0 1px 3px rgba(0,0,0,.1);';
    btn.onclick = () => {
      if (group.getLayers().length) {
        map.fitBounds(group.getBounds().pad(0.12));
      } else {
        map.setView([46.5, 2.5], 6); // fallback France
      }
    };
    return btn;
  };
  zoomCtrl.addTo(map);
</script>
<!-- ====== FIN CARTE ====== -->

<!-- ====== LISTE + RECHERCHE/FILTRES ====== -->
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

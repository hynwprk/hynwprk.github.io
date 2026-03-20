---
layout: page
title: cities
permalink: /cities/
description: photos from cities I've lived in
nav: false
images:
  lightbox2: true
---

<div class="cities-grid">
{% for city in site.data.cities %}
{% if city.photos.size > 0 %}
<div class="city-card" id="{{ city.slug }}">
  {% for photo in city.photos %}
  <a href="{{ '/assets/img/cities/' | append: city.slug | append: '/' | append: photo | relative_url }}" data-lightbox="{{ city.slug }}" data-title="{{ city.name }}">
    <img src="{{ '/assets/img/cities/' | append: city.slug | append: '/' | append: photo | relative_url }}" alt="{{ city.name }}" loading="lazy" />
  </a>
  {% endfor %}
  <span class="city-label">{{ city.name }}</span>
</div>
{% endif %}
{% endfor %}
</div>

<style>
.cities-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 16px;
  margin-top: 1rem;
}
.city-card {
  position: relative;
  overflow: hidden;
  border-radius: 8px;
  aspect-ratio: 4 / 3;
}
.city-card a {
  display: block;
  width: 100%;
  height: 100%;
}
.city-card img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
  transition: transform 0.4s ease, filter 0.4s ease;
}
.city-card:hover img {
  transform: scale(1.05);
  filter: brightness(1.05);
}
.city-label {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  padding: 8px 14px;
  font-size: 0.95rem;
  font-weight: 500;
  color: #fff;
  background: linear-gradient(transparent, rgba(0,0,0,0.55));
  pointer-events: none;
  letter-spacing: 0.02em;
}
@media (max-width: 576px) {
  .cities-grid {
    grid-template-columns: 1fr;
  }
}
</style>

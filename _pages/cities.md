---
layout: page
title: cities
permalink: /cities/
description: memories
nav: false
images:
  lightbox2: true
---

<div class="cities-grid">
{% for city in site.data.cities %}
{% if city.photos.size > 0 %}
<div class="city-card{% if city.portrait %} portrait{% endif %}" id="{{ city.slug }}">
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
  columns: 3;
  column-gap: 12px;
  margin-top: 1rem;
}
.city-card {
  position: relative;
  overflow: hidden;
  border-radius: 6px;
  margin-bottom: 12px;
  break-inside: avoid;
}
.city-card a {
  display: block;
}
.city-card img {
  width: 100%;
  height: auto;
  display: block;
  transition: transform 0.4s ease, filter 0.4s ease;
}
.city-card:hover img {
  transform: scale(1.03);
  filter: brightness(1.05);
}
.city-label {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  padding: 10px 14px;
  font-size: 0.9rem;
  font-weight: 500;
  color: #fff;
  background: linear-gradient(transparent, rgba(0,0,0,0.5));
  pointer-events: none;
  letter-spacing: 0.02em;
}
@media (max-width: 768px) {
  .cities-grid {
    columns: 2;
  }
}
@media (max-width: 480px) {
  .cities-grid {
    columns: 1;
  }
}
</style>

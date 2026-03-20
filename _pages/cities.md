---
layout: page
title: cities
permalink: /cities/
description: photos from cities I've lived in
nav: false
images:
  lightbox2: true
---

{% for city in site.data.cities %}
{% if city.photos.size > 0 %}
<h2 id="{{ city.slug }}">{{ city.name }}</h2>
<div class="city-gallery">
  {% for photo in city.photos %}
  <a href="{{ '/assets/img/cities/' | append: city.slug | append: '/' | append: photo | relative_url }}" data-lightbox="{{ city.slug }}" data-title="{{ city.name }}">
    <img src="{{ '/assets/img/cities/' | append: city.slug | append: '/' | append: photo | relative_url }}" alt="{{ city.name }}" />
  </a>
  {% endfor %}
</div>
{% endif %}
{% endfor %}

<style>
.city-gallery {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 10px;
  margin-bottom: 2rem;
}
.city-gallery a {
  display: block;
  overflow: hidden;
  border-radius: 6px;
}
.city-gallery img {
  width: 100%;
  height: 200px;
  object-fit: cover;
  transition: transform 0.3s ease;
}
.city-gallery img:hover {
  transform: scale(1.05);
}
</style>

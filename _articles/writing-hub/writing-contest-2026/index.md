---
layout: base
title: Writing Contest 2026
date: 2026-03-26
permalink: /writing-hub/writing-contest-2026/
event_path_prefix: writing-hub/writing-contest-2026/
hide_from_home: true
---

{% assign event_posts = site.articles | sort: 'date' | reverse | where_exp: 'item', "item.path contains page.event_path_prefix and item.hide_from_home != true" %}

<section class="hero">
  <div class="container">
    <h1 class="hero-title">Writing Contest 2026</h1>
    <p class="hero-subtitle">All posts submitted for this event</p>
  </div>
</section>

{% if event_posts.size > 0 %}
  {% assign featured = event_posts | first %}
  <section class="featured">
    <div class="container">
      <a href="{{ featured.url | relative_url }}" class="featured-card">
        {% if featured.image %}
        <div class="featured-image">
          <img src="{{ featured.image | relative_url }}" alt="{{ featured.title }}">
        </div>
        {% else %}
        <div class="featured-image featured-placeholder">
          <svg width="64" height="64" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
            <path d="M12 2L2 7l10 5 10-5-10-5zM2 17l10 5 10-5M2 12l10 5 10-5" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
          </svg>
        </div>
        {% endif %}
        <div class="featured-body">
          <div class="card-meta">
            <time datetime="{{ featured.date | date_to_xmlschema }}">{{ featured.date | date: "%d %b %Y" }}</time>
            {% if featured.tags.size > 0 %}
              <span class="card-tag">{{ featured.tags | first }}</span>
            {% endif %}
          </div>
          <h2 class="featured-title">{{ featured.title }}</h2>
          <p class="featured-excerpt">{{ featured.excerpt | strip_html | truncatewords: 50 }}</p>
          <span class="read-more">Read more →</span>
        </div>
      </a>
    </div>
  </section>

  {% assign rest = event_posts | slice: 1, 100 %}
  {% if rest.size > 0 %}
  <section class="articles-grid">
    <div class="container">
      <div class="grid">
        {% for post in rest %}
        <a href="{{ post.url | relative_url }}" class="article-card">
          {% if post.image %}
          <div class="card-image">
            <img src="{{ post.image | relative_url }}" alt="{{ post.title }}" loading="lazy">
          </div>
          {% else %}
          <div class="card-image card-placeholder">
            <svg width="40" height="40" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
              <path d="M12 2L2 7l10 5 10-5-10-5zM2 17l10 5 10-5M2 12l10 5 10-5" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
            </svg>
          </div>
          {% endif %}
          <div class="card-body">
            <div class="card-meta">
              <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%d %b %Y" }}</time>
              {% if post.tags.size > 0 %}
                <span class="card-tag">{{ post.tags | first }}</span>
              {% endif %}
            </div>
            <h3 class="card-title">{{ post.title }}</h3>
            <p class="card-excerpt">{{ post.excerpt | strip_html | truncatewords: 25 }}</p>
            <span class="read-more">Read more →</span>
          </div>
        </a>
        {% endfor %}
      </div>
    </div>
  </section>
  {% endif %}

  <section class="page-body">
    <div class="container container--narrow">
      <p>
      </p>
    </div>
  </section>
{% else %}
<section class="empty-state">
  <div class="container">
    <p>No posts in this event yet.</p>
  </div>
</section>
{% endif %}

---
layout: page
title: Writing Hub
date: 2026-03-26
permalink: /writing-hub/
---

<section class="hub-hero">
  <p class="hub-eyebrow">Community Program</p>
  <h2>Writing Hub 2026</h2>
  <p>
    A collaborative writing space where engineers, platform teams, and cloud native builders
    in Vietnam share practical lessons with the community.
  </p>
  <p>
    <strong>What We Are Looking For:</strong> hands-on Kubernetes and platform engineering
    experiences, case studies from production systems, GitOps, observability, security, cost
    optimization stories, and tooling deep-dives with actionable takeaways.
  </p>
  <p>
    <strong>How The Flow Works:</strong> pick a topic, create your post in this repository, open
    a Pull Request for organizer review, then publish after approval.
  </p>
  <p>
    <strong>Need Help?</strong> if this is your first contribution, use the
    <a href="{{ '/writing-hub/contribute-guide' | relative_url }}">Contribute Guide</a> for
    naming rules, front matter format, and the submission checklist.
  </p>
  <div class="hub-hero-cta">
    <a class="hub-btn hub-btn--solid" href="{{ '/writing-hub/contribute-guide' | relative_url }}">Read Contribute Guide</a>
  </div>
</section>

{% assign all_articles = site.articles | sort: 'date' | reverse %}

{% for event in site.data.writing_hub_events %}
  {% assign event_posts = all_articles | where_exp: 'item', "item.path contains event.path_prefix and item.hide_from_home != true" %}
  <section class="hub-event-block">
    <div class="hub-event-head">
      <h3>{{ event.title }}</h3>
    </div>

    {% if event_posts.size > 0 %}
    <div class="hub-event-grid">
      {% for post in event_posts limit:4 %}
      <a class="hub-event-card" href="{{ post.url | relative_url }}">
        <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%d %b %Y" }}</time>
        <h4>{{ post.title }}</h4>
        <p>{{ post.excerpt | strip_html | truncatewords: 20 }}</p>
      </a>
      {% endfor %}
    </div>
    {% else %}
    <p class="hub-empty">No posts in this event yet.</p>
    {% endif %}

    <div class="hub-event-cta">
      <a class="hub-btn hub-btn--outline" href="{{ event.url | relative_url }}">See more</a>
    </div>
  </section>
{% endfor %}

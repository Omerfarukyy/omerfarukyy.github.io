---
title: Security Research
icon: fas fa-shield-halved
order: 3
---

{% assign research = site.posts | where_exp: 'post', "post.categories contains 'Security Research'" %}

<section class="portfolio-page">
  <header class="page-intro">
    <span class="eyebrow">Responsible disclosure</span>
    <p>Public research, advisories, and proof-of-concept notes are published only after the relevant disclosure conditions are met.</p>
  </header>

  {% if research.size > 0 %}
    <div class="work-grid">
      {% for post in research %}
        {% include work-card.html item=post kind='research' label='Security research' %}
      {% endfor %}
    </div>
  {% else %}
    <div class="empty-state">
      <i class="fas fa-shield-halved" aria-hidden="true"></i>
      <p>No public research is listed yet.</p>
    </div>
  {% endif %}
</section>

---
title: Security Research
icon: fas fa-shield-halved
order: 3
---

{% assign research = site.posts | where_exp: 'post', "post.categories contains 'Security Research'" %}

<section class="portfolio-page">
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

---
title: Projects
icon: fas fa-cubes
order: 1
---

{% assign projects = site.posts | where_exp: 'post', "post.categories contains 'Projects'" %}

<section class="portfolio-page">
  <header class="page-intro">
    <span class="eyebrow">Selected work</span>
    <p>Tools, experiments, and open-source work. Each entry links to a short case study and its source when available.</p>
  </header>

  {% if projects.size > 0 %}
    <div class="work-grid work-grid--full">
      {% for project in projects %}
        {% include work-card.html item=project kind='project' label='Project' %}
      {% endfor %}
    </div>
  {% else %}
    <div class="empty-state">
      <i class="fas fa-cubes" aria-hidden="true"></i>
      <p>Projects will appear here as they are ready to share.</p>
    </div>
  {% endif %}
</section>

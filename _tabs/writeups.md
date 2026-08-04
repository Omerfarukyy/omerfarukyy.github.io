---
title: Writeups
icon: fas fa-terminal
order: 2
---

{% assign htb_boxes = site.posts | where_exp: 'post', "post.categories contains 'HTB Boxes'" %}
{% assign htb_academy = site.posts | where_exp: 'post', "post.categories contains 'HTB Academy'" %}
{% assign thm = site.posts | where_exp: 'post', "post.categories contains 'TryHackMe'" %}

<section class="portfolio-page writeups-page">
  <section class="work-section work-section--primary" id="htb-boxes">
    <header class="work-section__heading">
      <div>
        <h2>HTB Boxes</h2>
      </div>
      <span>{{ htb_boxes.size }} writeups</span>
    </header>
    {% if htb_boxes.size > 0 %}
      <div class="work-grid work-grid--preview">
        {% for post in htb_boxes limit: 3 %}
          {% include work-card.html item=post kind='htb' label='HTB Box' %}
        {% endfor %}
      </div>
      {% if htb_boxes.size > 3 %}
        <div class="view-all-row">
          <a href="{{ '/categories/htb-boxes/' | relative_url }}" class="view-all-link">View all {{ htb_boxes.size }} writeups <i class="fas fa-arrow-right" aria-hidden="true"></i></a>
        </div>
      {% endif %}
    {% else %}
      <div class="empty-state"><p>HTB Box writeups will be collected here.</p></div>
    {% endif %}
  </section>

  <section class="work-section work-section--academy" id="htb-academy">
    <header class="work-section__heading">
      <div>
        <h2>HTB Academy</h2>
      </div>
      <span>{{ htb_academy.size }} notes</span>
    </header>
    {% if htb_academy.size > 0 %}
      <div class="work-grid work-grid--preview">
        {% for post in htb_academy limit: 3 %}
          {% include work-card.html item=post kind='academy' label='HTB Academy' %}
        {% endfor %}
      </div>
      {% if htb_academy.size > 3 %}
        <div class="view-all-row">
          <a href="{{ '/categories/htb-academy/' | relative_url }}" class="view-all-link">View all {{ htb_academy.size }} notes <i class="fas fa-arrow-right" aria-hidden="true"></i></a>
        </div>
      {% endif %}
    {% else %}
      <div class="empty-state"><p>HTB Academy notes will be collected here.</p></div>
    {% endif %}
  </section>

  <section class="work-section work-section--thm" id="tryhackme">
    <header class="work-section__heading">
      <div>
        <h2>TryHackMe</h2>
      </div>
      <span>{{ thm.size }} notes</span>
    </header>
    {% if thm.size > 0 %}
      <div class="work-grid work-grid--preview">
        {% for post in thm limit: 3 %}
          {% include work-card.html item=post kind='thm' label='TryHackMe' %}
        {% endfor %}
      </div>
      {% if thm.size > 3 %}
        <div class="view-all-row">
          <a href="{{ '/categories/tryhackme/' | relative_url }}" class="view-all-link">View all {{ thm.size }} notes <i class="fas fa-arrow-right" aria-hidden="true"></i></a>
        </div>
      {% endif %}
    {% else %}
      <div class="empty-state"><p>Additional TryHackMe notes will appear here when useful.</p></div>
    {% endif %}
  </section>
</section>


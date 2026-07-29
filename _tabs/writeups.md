---
title: Writeups
icon: fas fa-terminal
order: 2
---

{% assign htb_boxes = site.posts | where_exp: 'post', "post.categories contains 'HTB Boxes'" %}
{% assign htb_academy = site.posts | where_exp: 'post', "post.categories contains 'HTB Academy'" %}
{% assign thm = site.posts | where_exp: 'post', "post.categories contains 'TryHackMe'" %}

<section class="portfolio-page writeups-page">
  <header class="page-intro">
    <span class="eyebrow">Labs and techniques</span>
    <p>Practical notes organised by platform, with tags for subjects such as Active Directory, web exploitation, Linux, Windows, and more.</p>
  </header>

  <section class="work-section work-section--primary" id="htb-boxes">
    <header class="work-section__heading">
      <div>
        <span class="eyebrow">Primary focus</span>
        <h2>HTB Boxes</h2>
      </div>
      <span>{{ htb_boxes.size }} writeups</span>
    </header>
    {% if htb_boxes.size > 0 %}
      <div class="work-grid">
        {% for post in htb_boxes %}
          {% include work-card.html item=post kind='htb' label='HTB Box' %}
        {% endfor %}
      </div>
    {% else %}
      <div class="empty-state"><p>HTB Box writeups will be collected here.</p></div>
    {% endif %}
  </section>

  <section class="work-section work-section--academy" id="htb-academy">
    <header class="work-section__heading">
      <div>
        <span class="eyebrow">Structured learning</span>
        <h2>HTB Academy</h2>
      </div>
      <span>{{ htb_academy.size }} notes</span>
    </header>
    {% if htb_academy.size > 0 %}
      <div class="work-grid">
        {% for post in htb_academy %}
          {% include work-card.html item=post kind='academy' label='HTB Academy' %}
        {% endfor %}
      </div>
    {% else %}
      <div class="empty-state"><p>HTB Academy notes will be collected here.</p></div>
    {% endif %}
  </section>

  <section class="work-section work-section--thm" id="tryhackme">
    <header class="work-section__heading">
      <div>
        <span class="eyebrow">Additional labs</span>
        <h2>TryHackMe</h2>
      </div>
      <span>{{ thm.size }} notes</span>
    </header>
    {% if thm.size > 0 %}
      <div class="work-grid">
        {% for post in thm %}
          {% include work-card.html item=post kind='thm' label='TryHackMe' %}
        {% endfor %}
      </div>
    {% else %}
      <div class="empty-state"><p>Additional TryHackMe notes will appear here when useful.</p></div>
    {% endif %}
  </section>
</section>

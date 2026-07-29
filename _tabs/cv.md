---
title: CV
icon: fas fa-file-lines
order: 4
---

<section class="cv-page">
  <header class="page-intro">
    <span class="eyebrow">Curriculum vitae</span>
    <p>This browser-readable CV will contain the concise version of experience, education, skills, and selected work.</p>
  </header>

  <div class="cv-format-card">
    <div>
      <span class="work-card__label">Available format</span>
      <h2>Online CV</h2>
      <p>Use this page for the readable, always-current version of the CV.</p>
    </div>
    <span class="format-label">HTML</span>
  </div>

  {% if site.cv.pdf_url %}
    <p><a class="action-link action-link--primary" href="{{ site.cv.pdf_url | relative_url }}">Download CV <span>PDF</span></a></p>
  {% else %}
    <p class="empty-copy">A PDF download will be added here once the final document is ready.</p>
  {% endif %}
</section>

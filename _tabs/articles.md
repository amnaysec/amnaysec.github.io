---
layout: page
title: Articles
icon: fas fa-newspaper
order: 2
---

{% assign standalone_articles = site.posts | where: "article", true %}

<p class="lead">
  Technical insights, industry news, threat analysis, and personal reflections on cybersecurity, networking, and cloud architectures.
</p>

{% if standalone_articles.size == 0 %}
  <p>No articles published yet.</p>
{% else %}
  <div id="articles-list" class="mt-4">
    <ul class="list-group list-group-flush">
      {% for post in standalone_articles %}
        <li class="list-group-item bg-transparent pl-0 pr-0 py-3 border-bottom">
          <div class="d-flex w-100 justify-content-between align-items-center">
            <h2 class="h5 font-weight-bold mb-1">
              <a href="{{ post.url | relative_url }}" class="text-decoration-none">
                {{ post.title }}
              </a>
            </h2>
            <small class="text-muted text-nowrap ml-2">{{ post.date | date: "%b %d, %Y" }}</small>
          </div>
          
          {% if post.description %}
            <p class="text-muted small mb-2 mt-1">{{ post.description }}</p>
          {% endif %}

          <div class="d-flex align-items-center small text-muted">
            {% if post.categories.size > 0 %}
              <span class="mr-3">
                <i class="far fa-folder mr-1"></i>
                {{ post.categories | join: ", " }}
              </span>
            {% endif %}

            {% if post.tags.size > 0 %}
              <span>
                <i class="fas fa-tags mr-1"></i>
                {{ post.tags | join: ", " }}
              </span>
            {% endif %}
          </div>
        </li>
      {% endfor %}
    </ul>
  </div>
{% endif %}
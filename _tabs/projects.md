---
layout: page
title: Projects
icon: fas fa-project-diagram
order: 1
---

{% assign project_posts = site.posts | where_exp: "item", "item.project != nil" %}
{% assign grouped_projects = project_posts | group_by: "project" %}

<p class="lead">
An overview of my technical projects and hands-on work.
</p>

{% if grouped_projects.size == 0 %}
  <p>Aucun projet documenté pour le moment.</p>
{% else %}
  <div id="projects-list" class="mt-4">
    {% for project in grouped_projects %}
      <div class="card mb-4">
        <div class="card-body">
          <h2 class="card-title h4 font-weight-bold text-primary mb-3">
            <i class="fas fa-folder-open mr-2"></i>{{ project.name }}
          </h2>
          <ul class="list-group list-group-flush">
            {% for post in project.items %}
              <li class="list-group-item bg-transparent pl-0 pr-0">
                <a href="{{ post.url | relative_url }}" class="font-weight-bold text-decoration-none">
                  {{ post.title }}
                </a>
                <span class="text-muted small ml-2">— {{ post.date | date: "%b %d, %Y" }}</span>
                {% if post.description %}
                  <p class="text-muted small mb-0 mt-1">{{ post.description }}</p>
                {% endif %}
              </li>
            {% endfor %}
          </ul>
        </div>
      </div>
    {% endfor %}
  </div>
{% endif %}
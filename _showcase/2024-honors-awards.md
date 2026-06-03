---
show: false
title: "Honors & Awards"
date: 2024-01-12
width: 6
---
<div class="m-4">
    <h5>Honors &amp; Awards</h5>
    <ul class="list small pl-3 mb-1">
        {% for item in site.data.profile.awards %}
        <li>
            <div class="d-flex">
                <div>{{ item.name }}</div>
                <div class="ml-auto mt-auto no-break"><em>{{ item.date }}</em></div>
            </div>
        </li>
        {% endfor %}
    </ul>
</div>

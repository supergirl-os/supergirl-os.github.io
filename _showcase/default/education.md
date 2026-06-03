---
show: true
width: 12
date: 2024-12-01
---
<div class="m-4">
    <h3>Education &amp; Honors</h3>
    <div class="row">
        <div class="col-lg-6">
            <h5>Education</h5>
            <ul class="list-unstyled mb-1">
                {% for item in site.data.profile.education %}
                <li class="media mb-1">
                    <img src="{{ item.logo | relative_url }}" alt="{{ item.name }}" style="width: 18px;" class="mr-1 mt-1">
                    <div class="media-body">
                        <div>{{ item.name }}</div>
                        <div class="small">{{ item.dept }}</div>
                        <div class="small d-flex">
                            <div>{{ item.position }}</div>
                            <div class="mt-auto ml-auto no-break"><em>{{ item.date }}</em></div>
                        </div>
                    </div>
                </li>
                {% endfor %}
            </ul>
        </div>
        <div class="col-lg-6">
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
    </div>
</div>

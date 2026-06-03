---
show: true
width: 4
date: 2024-12-02
height: 495px
images:
- src: /assets/images/etc/view.png
  title: Jiuzhaigou Valley
  # desc: Description 1.
  # link: https://picsum.photos/
- src: /assets/images/etc/basket.png
  title: Basketball
  # desc: Description 2
- src: /assets/images/etc/test.png
- src: /assets/images/etc/zion.png
  title: Zion National Park
- src: /assets/images/etc/tahoe.png
---

<div class="p-4 pb-0">
  <h3>Album</h3>
</div>
{% include widgets/carousel.html id=page.id images=page.images height=page.height %}
